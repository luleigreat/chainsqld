# 账本落盘与重放同步方案

本文整理启动扫树过慢、共识块丢失、以及整本拉取依赖 FullBelowCache 的问题，并给出分阶段改造方案。目标是：启动后即可参与共识、valid 账本不丢、补块不再绑整棵状态树。

---

## 1. 背景

用户数据量很大（其中一个合约发行了约 1400 万 NFT）。节点启动后，在磁盘读取打满的情况下，`checkLoadLedger` 仍需约 3 小时才能走完。这段时间内共识已经通过的区块容易丢失。

对比以太坊：日常启动是打开已有状态库继续执行，不把「整棵状态树 walk 齐」当作每个块、每次启动的完成条件。出块是对父状态执行交易并校验 `stateRoot`，而不是把子块整棵 trie 再拉一遍。

---

## 2. 要解决的问题

### 2.1 启动扫树期间丢失 validLedger

`checkLoadLedger` 完成前，共识已经认可的 `validLedger` 必须全部写入 RocksDB + SQLite，`complete_ledgers` 必须跟着往前走。

### 2.2 区块同步过慢、节点长时间不可用

当前补块路径要求拉到「完整账本」（头 + state 树 + tx 树 + 合约 storage）。这依赖 `checkLoadLedger` 预加载来填充 FullBelowCache。该过程本身极慢；即使走完，FullBelowCache 也会逐渐过期。之后再出现需要同步的情况，依然很慢，节点长时间无法参与共识。

### 2.3 对话中已确认的其它问题

这些和上面两点同源，方案里一并处理：

| 现象 | 原因 |
|---|---|
| `validated` 往前、`published` / `complete_ledgers` 停住 | `doValid` 只换 tip，不 `setFullLedger`、不写 SQL |
| 未重启也会丢中间本 | 关账结果只进 RAM / 2 分钟 cache，tip 一换就掉 |
| inbound `Want:` + timeout | 中间本没落盘，对端同样给不出页 |
| `tryFill` 把历史标成 complete | 只认 SQL 头 + 父 hash 链，不 walk 树，区间虚高 |
| `walkLedger` 查不了合约树 | 只扫 state + tx，1400 万 NFT 的 storage 不在其检查范围，Inbound 却要齐 |
| `jtLEDGER_DATA` 上限 2 | `checkLoadLedger` / history / 子本 inbound 互相饿死 |
| 启动 load 跳过 `walkLedger` | 先把 `validated = published = load 的那本`，再异步扫树 |

---

## 3. 根因

启动 load 的那本树页通常已经在 RocksDB 里。`checkLoadLedger` 从根 `walk` / `acquire`，1400 万 NFT 的 storage 会把磁盘打满。这 3 小时多半是在**扫盘上已有的数据给 FullBelow 加热**，不是在下载。

与此同时：

- `doValid` 只换 `mValidLedger`，不 `setFullLedger`，不写 `Ledgers` / `hotLEDGER`
- `switchLCL` 在非 standalone 下也不落库
- 中间本只靠 RAM，2 分钟可被 evict
- `published` / `complete_ledgers` 还要等连续 inbound

FullBelow 只优化 `getMissingNodes`（整本拉取），默认约 **2 分钟**过期。执行交易走 `fetchNodeFromDB`，**不依赖** FullBelow。扫 3 小时给 inbound 加热，热完很快就凉，后面再 inbound 一样慢。

结论：当前把「树扫齐 + FullBelow」当成继续共识和补块的前提，而不是「父本能执行、子本 Root 对得上、立刻落盘」。

---

## 4. 原则（用来卡住新 Bug）

1. **状态转移只走现有 `buildLedger`**（含 `updateSkipList`、统计、`ContractHelper.apply`、合约 storage）。不要另写一套 interpreter。
2. **只有 `accountHash` / `txHash`（及头里其它字段）和网络一致才 `setFullLedger`。** 对不上就失败，禁止标 complete。
3. **`complete_ledgers` 只在头 + 树页已进 NodeStore、SQL 头已写之后前进。** 禁止再靠 `tryFill` 只认 SQL 头就扩区间。
4. **共识线程不 `walk` 整树。** 落盘用已经 `flushDirty` 的脏页 + 写头。
5. **整本 InboundLedger 只作回退**（Root 对不上、缺页补不齐、父本不在盘上）。
6. **分阶段上线，每阶段可单独回滚。**

---

## 5. 分阶段方案

### 阶段 0（先做，风险最低）：关账即落盘

解决 2.1，也消掉「`checkLoadLedger` 期间丢块」。不改同步模型。

在已经 `buildLCL` + `checkLedgerAccept` 成功之后（`doValid` / `doValidLedger`），对**本机刚构出来的那本**立刻：

1. `LedgerHistory::insert(ledger, true)`
2. 写 NodeStore `hotLEDGER` 头（树页在 `buildLCL` 里已经 `flushDirty`）
3. `pendSaveValidated`（`Ledgers` + `Transactions`）
4. `mCompleteLedgers.insert(seq)`

把现在 `setFullLedger` 的落盘部分从「publish 成功」挪到「valid 成功」。`setPubLedger` / `pubLedger` / TableSync **仍可按序号往后发**，不要和落盘绑死。

注意：

- 用**同一份** `built.ledger_`，不要再 `getLedgerByHash`（可能已 evict）。
- SQL 交易表可异步，但 **头 + complete + History 必须在 `doValid` 返回前写完**，避免进程被杀仍丢。
- `pendSaveValidated` 若因缺 tx 节点失败，保持现有 `failedSave`：摘 complete、再 acquire，不要假装成功。关账本在内存里，一般不应失败。
- watching / non-validate 同样走 `buildLCL`，同一条路径。

做完后：启动扫树 3 小时，新块仍会进 RocksDB + SQLite，`complete_ledgers` 跟着涨。对端也能按 hash 供头。

### 阶段 1：拆掉启动对 checkLoadLedger 的依赖

默认不再为了「能共识」去 walk 整棵尖。

| 场景 | 做法 |
|---|---|
| 正常重启（load 出头，RocksDB 在） | 不跑 `checkLoadLedger`；打开账本当父本继续 `buildLedger` |
| 执行碰到 `SHAMapMissingNode` | 只拉**这一个 hash**（或该路径），重试该 tx；不要改成整本 acquire |
| 管理员 / 怀疑盘坏 | RPC 后台 `verify`，低优先级，**禁止** `acquire` 当前 tip，**禁止**占 `jtLEDGER_DATA` |

`onLastFullLedgerLoaded` 仍可把 `valid` / `pub` 设为 load 的那本。`tryFill` **不要**再把未 walk 的历史整段标进 `complete_ledgers`（或只给 history 回填用，不对外当「树齐」）。

共识可用性只要求：父本根在、执行路径能从 NodeStore 取出。未碰到的 NFT 叶子不必进内存，也不必进 FullBelow。

### 阶段 2：落后补块改为「拉头 + 交易树，在父本上 buildLedger」

与现有代码对齐，而不是新状态机。库里已有 `buildLedger(parent, txs, …)` 和 `LedgerReplay`。共识关账已经是这条路；缺的是 **同步也走它**。

落后 `pub+1 … valid` 时：

1. 用 tip 的 skip 列表拿下一本 **hash**（现有 `hashOfSeq`）。
2. 向 peer 只要 **header + tx SHAMap**（交易树很小）。
3. 父本用本地已落盘的 `seq-1`（阶段 0 之后应在盘上）。
4. `buildLedger(parent, txs, closeTime, …)`。
5. 比较算出的 `accountHash` / `txHash` / ledger hash 与头。一致 → 走阶段 0 同一套落盘 + `complete++`。
6. 不一致或父本不存在 → **一次** InboundLedger 整本拉取（旧路径），成功后再回到重放。

空块几乎只改 skip / 统计，和 1400 万 NFT 无关。有交易时只 descend 碰到的账户 / storage 槽。

`SHAMapMissingNode`：向 peer `get` 该节点写入 NodeStore，再执行。这比从根 `getMissingNodes` 扫整棵树小几个数量级，也 **不需要** FullBelow。

FullBelow 以后只服务「回退 inbound」，可以保持 2 分钟，不再当同步前提。

### 阶段 3（可选）：Inbound 降级

阶段 0+2 之后，Inbound 应很少走到。再考虑：

- `findNewLedgersToPublish` 优先 `getLedgerByHash`（盘上有）再重放，最后才 acquire
- `jtLEDGER_DATA` 不必为了可用性再加并发（加并发容易和写盘打架）
- `MAX_LEDGER_GAP = 100` 跳 published 先别动，避免 TableSync 以为连续

**共识缺 LCL 仍走 `Reason::CONSENSUS` 整树，不在本阶段 0–2 范围内。** 专项方案见 [ConsensusLedgerReplay.md](ConsensusLedgerReplay.md)：三处 CONSENSUS acquire 改 `acquireForConsensus`，父本在则重放，父本不在则等顺序补块，禁止对 tip 整本拉取。

---

## 6. 方案对比

| 做法 | 启动 3h | 丢 valid | 以后再同步 | 新 Bug 面 |
|---|---|---|---|---|
| 加长 FullBelow / 加快 walk | 仍要扫 1400 万 | 扫的时候仍丢 | cache 还会凉 | 低，但不解决问题 |
| 只做阶段 0 | 仍慢，但不丢块，complete 往前 | 解决 | 仍可能 inbound 慢 | **最低** |
| 阶段 0+1+2 | 启动即可共识 | 解决 | 重放，不依赖 FullBelow | 中（Root 校验能兜住） |
| 先改 inbound 只拉 delta、不落盘 | 仍丢 | 不解决 | 父本不齐就假增量 | 高 |

先做阶段 0，再关 `checkLoadLedger`，再切重放。不要三步合成一个 PR。

---

## 7. 必须防的新 Bug

1. **另写执行逻辑** → 和验证节点状态分叉。只调用现有 `buildLedger`。
2. **Root 不对仍标 complete** → 对外谎称有树，对端 inbound 再洞。对不上就失败回退。
3. **父本 hash 不是网络的 parentHash** → 禁止重放。
4. **`complete_ledgers` 出现空洞还对外说连续** → RangeSet 会显示 `0850,0855`。阶段 0 每本 valid 都落盘则不应出现。若有洞，`prevMissing` 只重放洞，不要 `tryFill` 填假连续。
5. **共识线程同步 walk / 大 SQL** → 出块超时。只同步写头 + complete；交易索引可异步。
6. **缺页时整本 acquire 当前 tip** → 又和 `checkLoadLedger` 一样把 NodeStore 打满。只拉 miss 的 hash。
7. **TableSync 以为 published 连续暴涨** → published 仍顺序；只让 complete / 磁盘先走。
8. **watching 空 `valPublic`**（已修）不要在落盘路径里再造一遍 ProposeSet。

---

## 8. 建议验证

- 重启 + 人为把 `checkLoadLedger` 卡住：期间出 N 本，重启后 RocksDB 有 N 个头，SQLite `Ledgers` 有 N 行，`complete_ledgers` 含这 N 个；进程被杀也不丢。
- 1400 万 NFT 合约：**不触达**的块，重放耗时应与 NFT 总量无关。
- 转一张 NFT：只看到少量 NodeStore 读，Root 与验证节点一致。
- 故意删掉某条未使用的 NFT leaf：共识仍能走；只有碰到该 key 才拉页或失败。
- Root 对不上：不进 complete，打日志，回退 inbound，不崩。
- 1 验证 + 3 tracking：全部关账即落盘，再人为落后，只靠头 + tx 重放追上。
- TableSync / 订阅：published 仍连续，不跳号。

---

## 9. 落地顺序

1. **阶段 0**（先做）：`doValid` 落盘 + complete 前进。直接消掉「扫树期间丢块」。
2. **阶段 1**：默认关掉启动 `checkLoadLedger`；缺页按 hash 补。启动后应能马上参与共识。
3. **阶段 2**：落后补块改 `buildLedger` 重放，Inbound 仅回退。同步不再绑 FullBelow。

阶段 0 不依赖重放也能单独上车；1+2 才是「不对整棵子树、只对 Root」。阶段 0 改动面小，和现有 `setFullLedger` / `buildLedger` 同语义，最不容易出新洞。
