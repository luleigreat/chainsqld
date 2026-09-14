# 共识缺 LCL：禁止整本 Inbound，改走重放

本文是 [LedgerPersistAndReplaySync.md](LedgerPersistAndReplaySync.md) 的后续。阶段 0–2 已经做到：valid 即落盘、启动不扫整树、`findNewLedgersToPublish` 优先 `tryReplayLedger`。

**共识入口还没接上。** `Need consensus ledger` 仍走 `InboundLedger::Reason::CONSENSUS`，`checkComplete()` 要求 header + state + tx + 合约树齐。状态大时这一下就是几小时，节点停在 `wrongLedger`。

目标：共识要某一本 hash 时，**只拉头 + 交易树，在本地父本上 `buildLedger`**。整本 Inbound 只作冷启动 / Root 对不上的回退。

依赖：阶段 0 的 `persistValidated`、阶段 2 的 `tryReplayLedger` / `replayFromHeaderTx`。不要另写执行器。

---

## 1. 背景

生产状态很大（单合约约 1400 万 NFT）。日志形态：

```
LedgerConsensus:WRN Need consensus ledger <hash>
```

对应 `Adaptor::acquireLedger`：本地 `getLedgerByHash` 为空，enqueue `jtADVANCE` / `getConsensusLedger`，再

```
InboundLedgers::acquire(hash, 0, Reason::CONSENSUS)
```

`seq` 传 0。CONSENSUS 与 GENERIC 一样要齐整棵 SHAMap，和 HISTORY 补洞、LedgerCleaner 没有本质 I/O 差别。差别只在**谁触发、要哪一本**：共识要的是当前网络 LCL（经常是已经跳出去的 tip），不是 `pub+1`。

阶段 2 的重放只挂在 publish 顺序补块。共识这条入口会绕过它，对着 tip 再拉一遍子树。

---

## 2. 问题

### 2.1 共识缺本 = 整树拉取

`InboundLedger::checkComplete()`：

| Reason | 完成条件 |
|---|---|
| `REPLAY` | header + tx |
| `CONSENSUS` / `GENERIC` / `HISTORY` | header + tx + state + contracts |

CONSENSUS 完成还会 `checkLedgerAccept` → `doValid`。树没齐之前共识轮不到正确 LCL，`wrongLedger` 持续数小时。

### 2.2 tip 跳跃放大灾难

`acquireLedger` 只要 **一个 hash**。网络 LCL 可以比本地 valid 超前很多。父本不在时整本拉 tip = 把 1400 万叶子从 peer 再搬一遍。`Need consensus ledger` 换 hash，等于上一本没拉完又开下一本。

publish 路径是 `pub+1 … valid` 顺序重放，不会对 tip 开整树。共识路径没有这层约束。

### 2.3 还有两处同样的入口

| 位置 | 行为 |
|---|---|
| `Adaptor::acquireLedger` | `requestAcquireForConsensus` |
| `NetworkOPs` 切网 LCL | 同上 |
| `RCLValidationsAdaptor::acquire` | 同上 |
| `RpcaPopAdaptor::checkLedgerAccept` | **禁止** GENERIC 整树；同样 `requestAcquireForConsensus` |

只改 Adaptor，另外两处仍会打满磁盘。

### 2.4 和「完全不走 InboundLedger」的界限

交易树必须从网上要，仍用 InboundLedger，但 **Reason::REPLAY**（只要 header+tx）。禁止的是对共识 LCL 开 CONSENSUS/GENERIC 整树。TxSet（本轮交易集）继续走 `InboundTransactions`，本文不改。

---

## 3. 原则（卡住新 Bug）

沿用落盘文档 §4，共识入口额外钉死：

1. **状态转移只走现有 `buildLedger` / `replayFromHeaderTx`。** 禁止为共识再写 interpreter。
2. **只有算出的 ledger hash（含 accountHash / txHash）与网络一致才 `persistValidated`。** 对不上失败，禁止标 complete、禁止当 LCL。
3. **父本 `parentHash` 必须等于网络头里的 parent。** 对不上禁止重放。
4. **共识线程禁止同步 walk 整树、禁止同步大 SQL、禁止同步 `buildLedger` 长路径。** `acquireLedger` 保持异步：没好就返回 `none`，下轮再问（和现在等 inbound 的形状一样）。
5. **父本不在本地时，禁止对 tip 开 CONSENSUS/GENERIC。** 返回 `none`，维持 `wrongLedger`，交给 `doAdvance` 从 `pub+1` 顺序重放。追上后 `getLedgerByHash` 自然成功。
6. **整本 Inbound 只留冷启动或 Root/缺页回退。** 缺页只补该 hash，禁止再 acquire 当前 tip（落盘文档 §7.6）。
7. **`Reason::REPLAY` 的 inbound 不得被 GENERIC/CONSENSUS 当成完整账本。** 已有 `InboundLedgers::acquire` 守卫，不能拆掉。
8. **同一 hash 不得并存 REPLAY 与 CONSENSUS。** 已有 REPLAY 进行中时，CONSENSUS 调用不得升级成整树；已有 CONSENSUS 整树进行中时，本方案应停掉/忽略它，改走重放，避免双拉。
9. **TableSync / published 仍顺序。** 重放成功只走 `persistValidated`（complete + 头），不 `setPubLedger`。
10. **watching / 空 `valPublic` 路径不要在重放里再造 ProposeSet。**

---

## 4. 方案

### 4.1 目标状态机（`acquireLedger(hash)`）

```
getLedgerByHash(hash)
  有 → 直接返回（已 immutable）

解析 seq（见 4.2）
  失败 → 只启动 REPLAY 要头（或等下轮）；禁止 CONSENSUS
         头到了再走下面

从目标 hash 沿 header.parentHash 往回走。总距离不封顶（须能走过 527 / 1000）：
  记住整条 path（tip → 最老）。下一轮从最老继续，不从 tip 重扫
  每轮最多 256 步（本 job 预算）；撞预算一条 WRN：`walk budget local=… network=… walked=256 replayed=…`
  撞预算 / 一批重放满 32 本 → job 结束并 **自己再挂 jtADVANCE**，让 publish 有机会跑（避免 MAX_LEDGER_GAP 跳号）
  已缓存 header 不再每步打 `Skip … walk parent`
  每轮 touch path 上未失败的 inbound，避免 1 分钟 sweep
  failed inbound 要 erase 再 acquire，不能 touch 续命
  重放成功只 pop 最老一本，**同一 job 继续下一本**（不再每本 return 等 15s timer）
  `REPLAY` inbound 完成（header+tx）→ `onReplayInboundReady` 立刻续跑，不要干等共识 timer
  新 tip 先接到已有 path（parent 已在 path / 短距离 join）；接不上仍继续最老一本
  `curSeq < local` 或同高 hash 不同或 `valid+1` 的 parent ≠ 本地 → `walk wrong-chain` 停
  当前本已在本地 → 若是目标则返回；否则 pop 已齐后缀，从 path 下一本继续
  无 header → REPLAY 当前 hash（只要头+tx），记住 walk 位置，返回 none（等 inbound 完成回调）
  parentHash 已在本地 → tryReplayLedger(当前)（通常是 valid+1）
  已 complete 的 GENERIC 整本：storeLedger 使用，**不要 erase**
  未拿到 header+tx 的 GENERIC/CONSENSUS：erase 改 REPLAY
  重放 apply 与共识对齐：retry pass + LedgerAdjust::updateTxCount
  root mismatch 只打日志并结束本 job，下轮可再试（禁止永久拉黑）；成功才 erase inbound
  落后 tip 时 `shouldProposeConsensus()==false`，禁止 proposing / `buildLCL` 错 child
  重放成功只 `persistValidated`（valid）。closed/open 一起动：`checkUpdateOpenLedger` 里先 `switchLCL(valid)` 再重建 open=valid+1，保证 `open.parentHash == closed.hash`
  parent 不在 → 当前 = parentHash，push 进 path，继续走

共识线程只 getLedgerByHash + requestAcquireForConsensus（jtADVANCE）。
禁止 acquire(CONSENSUS)。不要指望 tryAdvance 的 pub+1…valid
（valid 落后网络时补不到 valid+1…tip）。

本地连可执行父状态都没有（新库 / load 失败）
  → 允许一次 GENERIC 或 CONSENSUS（冷启动例外，默认关）
```

`NetworkOPs` 切网 LCL、`RCLValidationsAdaptor::acquire` 复用**同一函数**（建议：`LedgerMaster::acquireForConsensus(hash)`），禁止三处各写一份 if。

### 4.2 只有 hash、没有 seq

现网调用是 `acquire(hash, 0, …)`。`tryReplayLedger` 需要 `seq > 1` 才能 `getLedgerBySeq(seq-1)`。

按顺序取 seq，取到就停：

1. 进行中的 REPLAY inbound 若已有 header → `ledger->info().seq`
2. 当前 valid / pub 的 skip 列表 `hashOfSeq`
3. validations / proposal 里该 hash 带的 seq（peer 多数指向的 prevLedger 常有序号）
4. SQLite `Ledgers` 按 hash 查（只有曾经落过盘才有）

都没有：先 `acquire(hash, 0, REPLAY)` 只为拿头，**仍然不要 CONSENSUS**。头到了下轮就能走 4.1。

禁止用「当前 valid.seq + 1」去猜——网络 LCL 可能跳号，猜错会重放到另一本。

### 4.3 异步与线程

- `acquireLedger` 只允许 `getLedgerByHash` + `requestAcquireForConsensus`（`jtADVANCE` / `replayConsensusLedger`）。
- `replayFromHeaderTx` / `persistValidated` 只在该 job 里跑，共识线程不 `buildLedger`。
- `requestAcquireForConsensus` 同时只挂一个 job；job 内连续重放最多 32 本后 yield 再挂；等 inbound 时由 `onReplayInboundReady` 续跑。
- 同一 hash 的 `acquiringLedger_` 只用于去重日志。
- 重放成功后 `inboundTransactions_.newRound(seq)` 保持现有 `acquireLedger` 成功分支。

### 4.4 回退（必须窄）

| 失败 | 允许 | 禁止 |
|---|---|---|
| 父本不在 | 沿 parentHash 往回 REPLAY，先重放离本地最近的一本 | 对 tip CONSENSUS；只靠 pub+1…valid |
| REPLAY 超时 / fail | 擦掉该 inbound，下轮再 REPLAY；连续失败再考虑回退 | 第一次失败就整本 tip |
| Root 不对 | 日志 + 不 persist；本 job 结束，下轮可再试（parent/cache 清了可能过）；不升 GENERIC | 把 headerTx 当完整账本 `doValid`；永久拉黑该 hash；200ms 死循环重试 |
| `SHAMapMissingNode` | 按 hash 补该节点后重试该本（阶段 1 语义） | `acquire(当前 tip, CONSENSUS)` |
| 本地无任何可执行状态 | **一次** 整本（冷启动） | 之后每轮再开整本 |
| REPLAY 进行中 | 等 | `acquire` 升级为 CONSENSUS（`InboundLedgers` 已拒绝把 REPLAY 结果交给 CONSENSUS，但不要并存两种 reason） |

冷启动判定要严：例如 load 出的 tip 头在、RocksDB 能 `fetchNode` 到 accountRoot，就不算「无父状态」。不要把「getLedgerByHash 暂时 miss」当成新库。

### 4.5 改动落点（建议一 PR 只做共识入口）

1. `LedgerMaster` 新增 `acquireForConsensus(hash)`，内部调 `tryReplayLedger`，**默认不** `acquire(..., CONSENSUS)`。
2. `Adaptor::acquireLedger` 改为调它；去掉直接 `Reason::CONSENSUS`。
3. `NetworkOPs` 切网 LCL 同样改调它；切过去之前仍要 `canBeCurrent` / `isCompatible`（用重放出来的完整 built，不是 REPLAY 半成品）。
4. `RCLValidationsAdaptor::acquire` 同样改调它。分析 preferred LCL 不需要合约树齐，有 built 即可。
5. `RpcaPopAdaptor::checkLedgerAccept` 不得 `acquire(GENERIC)`。
6. 日志（info/warn，便于对照现网）：
   - `Need consensus ledger` 保留，后面跟 `replay` / `wait-parent` / `cold-inbound`
   - 重放成功复用 `replayFromHeaderTx built`
   - 跳过整本：`Skip consensus inbound <hash> seq=…; no parent, wait sequential replay`
   - 本轮步数用尽：`Need consensus ledger <hash> walk budget local=… network=… walked=256 replayed=…`
   - 错链刹车：`Need consensus ledger <hash> walk wrong-chain local=… at <cur> seq=…`
   - 重放成功（WRN）：`replayFromHeaderTx built <seq> <hash>`

不改：`buildLedger`、`TxSet`、`published` / TableSync、`tryFill`、LedgerCleaner、HISTORY 的 peer 范围跳过。

### 4.6 落地顺序

1. **只改 Adaptor**，另外两处先打「若走 CONSENSUS 打 error 日志」观察（或同步改掉，避免漏网）。推荐三处一起切到 `acquireForConsensus`，避免漏拉。
2. 观察：`Need consensus ledger` 后应出现 `replay` 或 `wait-parent`，不应再出现长时间 `Want:` 状态树。
3. 冷启动整本仍保留，用明确日志区分。

可单独回滚：恢复三处 `acquire(..., CONSENSUS)` 即可，不回滚阶段 0–2。

---

## 5. 必须防的新 Bug

1. **共识线程同步重放整本交易 / walk 树** → 出块超时。重放进 job，acquire 只返回已完成结果。
2. **Root 不对仍 `doValid` / `switchLastClosedLedger`** → 分叉。必须 `built->info().hash == expected`。
3. **把 REPLAY 半成品（无 state 树）交给 CONSENSUS 调用方** → 下一轮 `buildLCL` 炸或状态空。已有 `InboundLedgers` 守卫；`switchLastClosedLedger` 只能拿 `replayFromHeaderTx` 的 built。
4. **父本不在却整本拉 tip** → 回到现在的几小时不可用。这是本方案的主禁令。
5. **用 valid.seq+1 猜测网络 LCL 的 seq** → 重放到错误高度。seq 必须来自头 / skip / validation。
6. **同一 hash REPLAY + CONSENSUS 双拉** → 磁盘和 `jtLEDGER_DATA` 互抢。先到的 REPLAY 不得被升级。
7. **重放成功却 `setPubLedger` 跳号** → TableSync 以为连续。只 `persistValidated`。
8. **冷启动例外过宽** → 每次 miss 都整本。例外只认「没有任何可执行父状态」。
9. **`canBeCurrent` / `isCompatible` 被绕过** → 切到敌对链或时间错乱的 LCL。切网路径保持这两项检查。
10. **watching 空 `valPublic` 回归** → 重放路径不要碰 ProposeSet / `gotTxSet` 的公钥复用。
11. **`complete_ledgers` 出现洞还对外说连续** → RangeSet 如实；顺序补洞，不 `tryFill` 假连续。
12. **缺一页就 acquire tip** → 与 `checkLoadLedger` 同类。只拉 miss 的 node hash。
13. **每个 job 只重放一本就 return，且 REPLAY 完成不续跑** → 追上 tip 附近后只拉新 tip header，最后几本永远不重放。必须 job 内连续重放 + inbound 完成回调 + 预算自挂。
14. **只推 valid、不切 closed，或只切 closed 不改 open** → `beginConsensus` 假定 `open.parentHash == closed.hash`。必须在 `checkUpdateOpenLedger` 里两者一起动：先 `switchLCL(valid)`，再重建 open。
15. **`RpcaPopAdaptor::checkLedgerAccept` 对 tip 开 GENERIC** → 每个新 tip `drop non-REPLAY`，和重放抢 inbound。改走 `requestAcquireForConsensus`。
16. **追上（或还在 replay）就 `proposing` / `doAccept`** → 本机先 `buildLCL` 出错 child，再 replay 同一 seq；`ContractHelper` 无锁，`jtACCEPT` 与 `jtADVANCE` 并发 `flushDirty`/`clearCache` 会把 `std::map` 写崩。落后 tip 时只 observing/wrongLedger；`preStartRound` 看 `shouldProposeConsensus()`（仅 `!replay`）；init 窗口不 propose；`InitAnnounce` 只用本地 `previousLedger_.id()`；切到网络 LCL 立刻 `handleWrongLedger`。
17. **`mConsensusReplayMismatch` 永久拉黑** → 同一 hash 换 parent / 清缓存后能过，却再也追不上。mismatch 只记日志并结束本 job，下轮重试。
18. **validation 先到、本机还在 `buildLCL`** → `jtACCEPT` 与 `jtADVANCE` 双 apply，同一 tx-set `account_hash` 对不齐（4232）。`setBuildingLedger` **只在真正 `buildLCL` 时**置位，不得在 `startRound` / `onClose` / `onCollectFinish` 提前挂整轮。`requestAcquireForConsensus` / `checkLedgerAccept(hash,seq)` 对 `seq==building`（或 seq 未知）defer；`RCLValidations::acquire` 走同一入口。
19. **本地票够了就 `switchLCL` 私有 child（3711）** → `MovedOn` 禁止用本地 `result_` `buildLCL`；该 seq 已有 quorum hash H 则 `Skip buildLCL; use/acquire quorum ledger`，禁止再算一遍。`buildLCL` 之后若 H 已是另一 hash：`Discard local buildLCL`，丢掉 closed 索引，清 apply 缓存，replay H。不要用 `validSeq+3` settle 挡 propose（空链 seq 不涨会一直 abnormal）。空闲空池仍走 `omitEMPTY`。
20. **`walk wrong-chain` 只停不修** → 错的 3711 写进 SQL，重启仍卡。检测到本机 tip 与网络 parent/hash 不一致时：丢掉该 seq、valid/closed 回到 parent、按网络 hash replay。`tryReplayLedger` 不得用 hash 对不上的 `getLedgerBySeq(seq-1)` 当父本。这是 C/D 漏掉时的后盾，不是唯一防线。

`ContractHelper` 的 dirty/state/SHAMap cache 用一把 `recursive_mutex` 罩住 `flushDirty` / `clearCache` / `setStorage` / `apply`。进入 wrongLedger 和 replay 追上 tip 时 `clearConsensusApplyCaches()`。
catch-up 结束：`finishConsensusReplay` 先清缓存，再 `mConsensusReplayActive=false`。wrong-chain 回收期间保持 replay pending，禁止 `buildLCL`。
`shouldProposeConsensus()` = `!consensusReplayPending()`。`preStartRound` 和 POP `phaseCollecting` 都看它。

---

## 6. 和现有阶段的关系

| 已做 | 本方案用到的 |
|---|---|
| 阶段 0 `persistValidated` | 重放成功后同一套落盘；父本应在盘上 |
| 阶段 1 不扫启动树 | 共识可用性 = 父本能执行，不是 FullBelow |
| 阶段 2 `tryReplayLedger` | 共识入口直接调用，不复制一份 |
| HISTORY 无 peer 则 skip | 不改；共识重放不走 HISTORY |

本方案是落盘文档阶段 3 的「共识侧」：Inbound 降级从 publish 扩展到 LCL 获取。

---

## 7. 验证

- 落后 1 本：`Need consensus ledger` 后应 `replayFromHeaderTx built`，无长时间 `Want:` 状态节点；随后能进共识。
- 落后 N 本（N < MAX_LEDGER_GAP）：只见 `wait-parent` + publish 顺序重放，**禁止**对 tip 出现 CONSENSUS inbound；追上后 Adaptor 直接 `getLedgerByHash`。
- 1400 万 NFT、空块或无关交易：重放耗时与 NFT 总量无关。
- 转一张 NFT：少量 NodeStore 读，Root 与验证节点一致。
- 故意 Root 不对：不 persist、不 switchLCL、不进 complete。
- 故意缺一页未使用 NFT：共识仍走；碰到该 key 才补页或失败；不得整本 tip。
- 重启后立刻 `Need consensus ledger`：有父本则重放，无父本则 wait-parent，不得先 CONSENSUS。
- 新库冷启动：允许一次整本，日志标明 `cold-inbound`；之后落后走重放。
- 1 验证 + 3 tracking：watching 不崩、不造空 `valPublic`。
- TableSync / 订阅：published 连续，不因共识重放跳号。
- 落后期间：`Entering consensus` 应 `replay=yes`，不得 `proposing`；可见 `Skip buildLCL`，不应再 `flushDirty` 红黑树崩溃。
- mismatch 后下轮可再 `replayFromHeaderTx`；日志含 parentSeq/parent/txs；`fail>0` 打 `Replay apply seq=… success=… fail=…`。
- InitAnnounce：`prevSeq` 与 `prevHash` 必须同属本地 `previousLedger_`，禁止本地 seq 配网络 tip hash。
- 追上后：replay 一结束即可 propose；不得再出现 `Propose after catch-up settle until seq=`。空闲空池仍可 `Empty transaction-set from self` 走 omitEMPTY，`server_status` 能进 `normal`。
- 本机 3711 与网络 tx 数/hash 不同：应出现 `Skip buildLCL; use/acquire quorum ledger` 或 `Discard local buildLCL` / `MovedOn acquire`，不得 `switchLCL` 2 笔那本；SQL 该 seq 不是私有 child。后盾仍是 `drop wrong-chain` + `rewind=` + `replayFromHeaderTx built`。
- 同一 tx-set 不得再 `LedgerHistory MISMATCH` 且 `account_hash` 不同（4232）：build 期间 `Defer consensus acquire`。
- 回滚：恢复三处 CONSENSUS acquire，阶段 0–2 行为不变。

---

## 8. 建议实现顺序

1. 抽出 `LedgerMaster::acquireForConsensus`（先实现 4.1 + 4.2 + 父本缺失不整本）。
2. Adaptor / NetworkOPs 切 LCL / RCLValidations 三处改调用。
3. 补日志与 `SHAMapMissingNode` 按 hash 补（若阶段 1 单 hash 尚未落地，本步失败则 none + 下轮重试，**仍不要 tip CONSENSUS**）。
4. 冷启动例外最后加，默认关，用显式条件打开。

每步可单独回滚。不要和 HISTORY / cleaner / `tryFill` 绑在同一个提交。
