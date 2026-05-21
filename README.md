# How I Built Arkloop with FlowChat

这是我在 [Arkloop](https://github.com/ArkloopAI/Arkloop) 中构建 FlowChat 功能时，与 AI 的真实对话记录。

下面的内容只保留了 **我的 prompt** 和 **AI 的输出**，中间的工具调用、思考过程等已移除。

---

## 调研

> 你可以看到我电脑的 Document 文件夹下有一个叫 Flow Chat 的项目。是这样的，我想把它的理念融进 Arkloop。首先我们来讲一下这是个什么项目。很好理解，看一张图就能理解。他把对话做成了一种线的形式。我想给它融进 Arkloop，是想代替 Arkloop 原有的类似于 Retry 啊、Fork 啊等语义，让它变得更加的直观化、更加的好用化。好，看起来很乐观是吧？我们 Arkloop 的后端也有 fork 的这个根本了，做起来可能会不难。但是我们往下讲，对于 FlowChat 来讲，它实实在在的把输入框放进了整个思维导图中。对于我们来说，我们该怎么做呢？我当时思考这个问题思考了很久，直到我后来做出了我们 Arkloop 的右侧面板，我们做出了 right panel，我发现这一点就会变得极其的简单。也就是，平常呢这种东西肯定是不显现的，对吧？如果说你要点 Retry、你要点 Fork 按钮，或者说你点了左侧的标题上侧，展开之后，可能会显示对话分支图什么的，对吧？那所以我需要你帮我调研，也就是我把这个项目融入进我的项目，需要多少的精力？需要怎么做？我们后端要进行一轮比较大的改动吗？或者说我们现在的后端已经完全能够支持这一点了。而前端怎么融比较好呢？目前想的是放到 right panel 专门做一页出来。确实是不难的，而且源码基本上也能照抄。你认为怎么样？工作量大不大？我们点击了这个最尾部的对话图，或者说点击了一条消息的时候，我们的 ChatView 会重新渲染到这个时候的状态。你能理解这一点吗？

能理解，而且方向是对的：把 Flow Chat 的"分支可视化"放进 Arkloop right panel，是一个前端主导、中等复杂度的改造；后端不需要一轮大重构。

关键判断：

Arkloop 后端已经有根基。`threads.parent_thread_id` / `branched_from_message_id` 已经存在，`POST /v1/threads/{id}:fork` 也已经能复制到指定消息为止。
但它现在是 thread 级 fork，不是 Flow Chat 那种 `message.parent_id` 级 DAG。Flow Chat 的核心是每条消息天然指向父消息，当前分支就是从选中消息往上找祖先链。

所以最短路径不是把 Arkloop 改成 message DAG，而是：

1. right panel 新增一个"分支图"tab。
2. 图里把当前 thread 的消息画成主线，把 fork 出来的 thread 作为从某条消息长出来的分支。
3. 点击某个 thread 分支尾部：直接 `navigate('/t/:threadId')`，复用现有 ChatView 加载链路。
4. 点击当前 thread 内某条消息：ChatView 用"视图切片"渲染到该消息为止。
5. 如果用户在这个切片状态继续发送：不要破坏原 thread，调用 `forkThread(threadId, messageId)` 创建新 thread，再在新 thread 发送/运行。

这正好能替代现在用户感知里的 Retry / Fork：
Retry 不再像"重试按钮"，而是"从这条用户消息长出一条新回答分支"。Fork 不再像抽象操作，而是图上一条真实分支。

需要补的后端很小，但有几个必要点：

- `ThreadResponse` 目前没有暴露 `branched_from_message_id`，前端无法把 fork 精确挂回消息节点。
- 需要一个"获取当前 thread 分支族谱"的 API，至少返回 root/ancestor/children threads，否则前端只能从普通 thread 列表里猜。
- message 响应最好暴露 `thread_seq`，做"渲染到某条消息状态"会更稳定。

前端主要工作：

- `RightPanelTab` 增加一个 branch 类型。
- `ChatView` 里生成 `ConversationBranchPanel` tab，插入 `rightPanelRenderedTabs`。
- 分支图用 React Flow + Dagre 重写 Flow Chat 的 nodes/edges/layout 思路，不直接搬 Vue 代码。
- `ChatView` 现在的 `MessageList` 直接读全局 messages，需要让它支持"可见消息切片"，或者在 MessageStore 增加 selected branch view。

我的建议：先做 thread-fork 可视化，不要第一步就把后端改成 message DAG。
Arkloop 现在的 fork 语义已经足够支撑第一版，而且更符合现有架构。真正要避免的是继续让 retry/edit 走"隐藏尾部"的语义；视觉化之后，用户更自然期待的是"产生新分支"，不是"覆盖旧历史"。

工作量判断：不算大后端工程，主要是前端中等偏上的状态和交互工程。
如果只做 right panel 分支图 + 点击切换 thread，比较轻；如果要点击任意消息后 ChatView 进入"checkout 状态"并继续生成新分支，就会多一层 ChatView/MessageStore 状态治理，但仍然不需要大改 worker。

---

> 所以我们仍然是以 thread 为主，只是说每条新的分支都会变成一个新的 thread 是吗？因为我认为后端变成 Message DAG 可能并不完全适合所有场景，它可能只适合 Chat，但是如果说 Agent 什么的话，它有点不太适合这个语义

是的，我建议仍然以 thread 为主。

更准确地说：

Arkloop 的运行单位还是 thread。每次从某条历史消息分叉，本质上创建一个新的 thread；这个新 thread 记录自己来自哪个 parent thread、来自 parent thread 的哪条 message。

也就是现在已有的模型继续成立：

```
Thread A
  message 1
  message 2
  message 3

从 message 2 分叉

Thread B
  parent_thread_id = Thread A
  branched_from_message_id = message 2
  messages = message 1, message 2 的复制快照，然后继续跑
```

前端 right panel 只是把它画成：

```
message 1 -> message 2 -> message 3
                 \
                  -> Thread B 后续消息
```

这比改成全局 Message DAG 更适合 Arkloop。

原因很简单：Arkloop 的 thread 不只是 Chat 的消息容器，它还是 agent 执行上下文。它绑定了 run、tool event、workspace、right panel 状态、文件、运行中断、SSE、标题、权限、分享、私密模式、工作目录等东西。Agent 场景里，"一次分支"通常不是只复制几条 chat message，而是复制一个执行上下文的入口。

Message DAG 更适合纯聊天产品，因为它的世界很干净：消息 A 的父亲是消息 B，当前分支就是一条祖先链。但 Arkloop 里消息背后有 run、工具调用、文件产物、side effects。把所有东西压成 `message.parent_id` 会让模型变窄，后面反而难受。

所以最稳的抽象是：

- **Thread** = 一个可继续执行的上下文
- **Message** = thread 内的线性历史
- **Fork** = 从某个 thread 的某条 message 创建新 thread
- **Branch Graph** = thread 之间的可视化索引，不是底层执行模型

这也解释了 right panel 为什么合适。主 ChatView 仍然展示"当前 thread 的线性执行历史"；right panel 负责展示"这个 thread 家族的分支拓扑"。点击某条分支，本质是切换 thread；点击当前 thread 内某条消息，可以进入"预分叉视图"，继续发送时再创建新 thread。

我会反对把后端整体改成 Message DAG。那是为了一个 UI 体验去重塑运行模型，收益不够，风险很大。更好的路线是：thread 级 fork 保持为后端真相，message 级节点只是前端表达和 fork 锚点。

---

> 那从我们现在这一步的思考，到我们完整接入，我们目前还差什么？还有差什么需要讨论的呢？那下一步我们要做的话，无非就是加个前端，然后思考一下前端放哪，然后改一下后端的事情嘛。还有什么需要讨论的吗？还是说我们可以直接出 plan 去，马上进入下一步执行？

可以进入 plan 了。现在还需要讨论的不是"大方向"，而是几个会影响实现语义的边界。

我认为还差这几件事：

**分支图的真实边界**
第一版建议只画"当前 thread 家族"：当前 thread、祖先 thread、直接/间接子 thread，以及它们挂载到哪条消息。不要一开始做全局所有 thread 图。

**点击消息后的语义**
建议定义成两种状态：
- 点击某个分支尾部或 thread 节点：切换到对应 thread。
- 点击当前 thread 的某条历史消息：ChatView 进入"截断预览状态"，只显示到这条消息；继续发送时自动 fork 新 thread，而不是改写当前 thread。

**Retry 的新语义**
我建议把 Retry 逐步收敛成"从该 user message 生成新分支"，而不是继续隐藏后续消息。旧的破坏式 retry 可以先不暴露在 UI 里，但后端 API 暂时不用急着删。

**图里展示到什么粒度**
第一版建议：主线消息节点 + fork thread 分支节点。不要把每个分支 thread 的所有消息都铺满 right panel，否则图很快炸。点进分支后再加载该 thread 的线性消息。

**后端最小补口**
- `ThreadResponse` 暴露 `branched_from_message_id`。
- 新增一个 thread branch family API，返回当前 thread 的祖先、子孙、分叉消息锚点。
- message response 暴露 `thread_seq`，用于稳定截断和排序。

**右侧面板入口**
我建议作为 right panel 的固定系统 tab，不作为普通 resource tab。标题可以很短，例如"分支"，图标用 GitBranch 类图标。

**状态持久化**
第一版只持久化 panel 是否打开、active tab、viewport/focus node 可以存在 localStorage。不要把分支关系缓存成本地真相。

我的判断：可以直接出 plan 并进入执行。
剩下这些不是需要继续辩论的大问题，可以在 plan 里定死第一版边界，避免越做越像重构整个会话系统。

下一步 plan 应该按这个顺序：
1. 后端补数据契约。
2. 前端加 branch panel tab 和分支图组件。
3. 接通点击 thread/message 的交互。
4. 把"从历史消息继续发送"改成自动 fork。
5. 补测试和最小验证。

---

> 哦哦哦，你认为需要我给你这些建议是吧？首先，一，分支图肯定是画所有，哦哦，你你，这这，我没听懂你的意思。它不应该和 FlowChat 一样，就是这种分离式的吗？虽然后端可能有一堆 thread 对吧？那这我们前端渲染肯定是以这种 prompt 以 fork 为主啊。第二，点击消息后的语义，我还是认为是只显示到那条消息，如果继续发送就是继续 fork 到新 thread。Retry 的话，对，我也是想新分支，因为可以直接接近 FlowChat 的这种感觉。图里展示力度还是，我建议和 FlowChat 一样，至少我们后端之后再改啊，对吧？然后我觉得没必要总是最小补口、最小切口，就是我们得想要有一个正确的架构，而不是说每天都是为了一个最小的路径去实现

对，你这个纠正是对的。我前面说"第一版只画当前 thread 家族"，表达得太保守了，而且容易把这个东西做成一个普通 fork list，而不是 FlowChat 那种体验。

我重新整理一下。

正确架构应该是：前端按 FlowChat 的方式画消息图；后端仍然以 thread 作为执行上下文。

也就是说，不是画成：

```
Thread A
  -> Thread B
  -> Thread C
```

而是画成：

```
message 1 -> message 2 -> message 3
                 \
                  -> message 2' -> message 3'
                 \
                  -> message 2'' -> message 3''
```

只不过底层每条分支可能属于不同 thread。

所以前端视角是 message graph，后端视角是 thread fork graph。这两个不冲突。

关键是要加一层"投影模型"：

```
Backend truth:
Thread + Message + ForkAnchor

Frontend projection:
FlowNode + FlowEdge + ActivePath
```

后端不需要真的改成 `messages.parent_id` DAG，但后端需要能告诉前端：

```typescript
type ConversationGraph = {
  root_thread_id: string
  active_thread_id: string
  threads: GraphThread[]
  messages: GraphMessage[]
  edges: GraphEdge[]
}
```

其中 edge 可以表达：

```typescript
type GraphEdge =
  | { kind: 'next_message', from_message_id: string, to_message_id: string }
  | { kind: 'fork', from_message_id: string, to_message_id: string, from_thread_id: string, to_thread_id: string }
```

这样前端就可以完全像 FlowChat 一样渲染，而不是被 thread 形状绑住。

你说"分支图肯定是画所有"，我现在理解为：画这个对话树里的所有消息分支，不是只画一个 thread 的主线，也不是只画 thread 节点。这是对的。否则就不像 FlowChat，只是一个分支目录。

那需要讨论的核心就变成这些：

**Conversation Graph 的范围**
我建议是"同一个 root thread 下的整个 fork tree"。也就是从最早的 root thread 开始，所有由它 fork 出来的 thread 都属于同一张图。不是账号下所有 thread 混在一起。

**消息节点身份**
这是最关键的问题。因为 Arkloop 现在 fork 会复制消息，所以同一条历史消息在不同 thread 里会有不同 message id。
如果直接用 message id 画图，会出现主干被复制多份。
正确做法应该引入一个稳定的逻辑身份，例如：

- `message_node_key`：对 fork 前复制出来的历史消息，它们应该投影成同一个图节点；fork 后新增的消息才是新节点。
- fork copy 的映射要持久化。现在 `id_mapping` 只在 fork 响应里返回，不持久化。要做完整 FlowChat 图，这不够。需要后端保存 old message -> copied message 的关系，或者更进一步保存 `origin_message_id` / `graph_node_id`。

**Retry 改语义**
Retry 不应该再隐藏后续消息。新的 retry 应该是：
1. 找到 user message；
2. 从它所在 thread 的该 message 创建 fork thread；
3. 在新 thread 里重新生成 assistant；
4. 图上表现为从该 user message 长出一个新 assistant 分支。

**点击消息后的 ChatView**
这个语义我们已经一致：只显示到那条消息。继续发送时 fork 新 thread。
这里不是简单 `slice(messages)`，因为点到的节点可能属于 graph 投影，需要解析到具体 thread/message。

**输入框是否进入图**
right panel 里可以做 FlowChat 式输入框，但 Arkloop 主 ChatView 的输入框仍然保留。
如果用户在图节点上输入，就是从该节点 fork；如果在主 ChatView 输入，就是沿当前 active thread 继续。

所以我现在的判断是：可以出 plan，但 plan 不能是"小补口"。应该是一个正式的 Conversation Graph 架构 plan。

后端不大重构成 Message DAG，但需要新增一层图语义：
- `threads/messages` 是存储与执行模型
- `conversation_graph` 是展示与分支操作模型

这才是干净的。
否则只暴露 `branched_from_message_id`，前端硬拼，会很快变成一堆猜测逻辑。

---

> 输入框不入图，我们和 FlowChat 不一样，我们有自己的 ChatView，他们主要是以直接以这个图为聊天方式的，所以这个是一个区别点

对，这个边界很关键：输入框不入图。Arkloop 不是把图当主聊天界面，而是把图当分支导航和分支操作面板。

所以架构应该改成这样：

- **ChatView** = 主执行界面
- **Right Panel / Conversation Graph** = 分支地图

图负责：
- 看完整消息分支结构
- 选择某个消息节点
- 让 ChatView 只渲染到该消息
- 从该消息触发 retry/fork 语义
- 切换到某条分支所在 thread

图不负责：
- 承担主输入框
- 直接替代 ChatView
- 在节点旁边内嵌输入
- 变成第二套聊天界面

这样更适合 Arkloop。因为 Arkloop 的 ChatView 还承载工具调用、artifact、workspace、right panel 资源、run 状态、SSE、排队输入、中断恢复等东西。FlowChat 的图是主界面；Arkloop 的图应该是导航层/结构层。

最终交互可以定成：
1. 用户平时在 ChatView 正常聊天。
2. 打开 right panel 的"分支图"。
3. 点击图中任意消息：ChatView 渲染该消息所在路径，截止到该消息。图中高亮当前 active path。
4. 此时用户在 ChatView 底部输入框继续发送：如果选中消息不是当前 thread 的最后一条消息，则自动 fork 新 thread。如果选中消息是当前 thread 的最后一条消息，则沿当前 thread 继续。
5. Retry 按钮：不再破坏当前 thread。从对应 user message 生成一个新分支 thread。
6. 点击图中的分支尾部：切换到对应 thread 的完整末端状态。

这样就不会把 Arkloop 变成 FlowChat 的复制品，而是把 FlowChat 最有价值的"对话分支可视化"融入 Arkloop 的执行模型。

所以我们现在已经能进入 plan。plan 的核心目标不是"把输入框搬进图"，而是：

**建立 Conversation Graph 投影层，让 right panel 能按消息节点展示 thread fork tree，并驱动 ChatView 的 active path。**

---

> 我们聊的足够充分能开始做了吗

可以开始。

当前关键边界已经清楚：
1. 后端仍以 thread 作为执行上下文。
2. 前端 right panel 渲染的是 Conversation Graph 投影，不是把后端改成 Message DAG。
3. 输入框不入图，仍由 ChatView 承担。
4. 点击图中消息后，ChatView 渲染到该消息。
5. 从历史消息继续发送时自动 fork 新 thread。
6. Retry 语义改成生成新分支，不再破坏当前 thread。
7. 图展示完整 fork tree 下的消息分支，而不是 thread 列表。

下一步应该直接进入执行计划，顺序是：
1. 定义 Conversation Graph 后端 contract。
2. 补后端 fork graph 查询能力和持久映射。
3. 前端加 right panel 分支图 tab。
4. 接通图节点选择到 ChatView active path。
5. 改发送和 Retry 的分支语义。
6. 补测试，验证 fork、点击历史消息、继续发送、切换分支全链路。

---

## Review & 修复

> review 这个 flowchat 会带出什么问题，也就是 review 这个 commit，特别不是他自己的 bug，而是他连带的稳定性问题

我会按 review 方式看这个 commit 的连带风险，不只看"图能不能画出来"，重点查它改动后对 thread 切换、消息加载、run 恢复、侧边栏和后端查询的稳定性影响。

重点风险集中在：Retry 语义改成 fork 后 run 状态链路、跨分支切换的 handoff 和右侧状态、后端递归树查询边界。

### Review 结论

这个 commit 最大的问题不是图本身，而是它把"branch graph"接进了 Arkloop 原有的 thread / run / SSE / sidebar 生命周期后，几个边界会互相污染。按我现在的判断，下面这些是需要优先修的。

**P1 必修**

`useChatActions.ts` (line 389) Retry 创建 fork run 后，先 `setActiveRunId(run.id)` 再 navigate 到 fork thread。
风险：SSE 会先在旧 thread 上启动，新 run 的 live UI、handoff、terminal 状态可能写到源 thread。
修法：Retry fork 后不要在旧 thread 上 `setActiveRunId`，只通过 navigation state 带 `initialRunId`，等新 thread 的 ChatView 接管后再启动。

`ChatView.tsx` (line 1307) Retry 后用 `graphFocusMessageId` 设置 `selectedGraphMessageId`，随后 `ChatView.tsx` (line 1904) 会截断消息列表。
风险：Retry 的 assistant 输出到达后会被截断隐藏，看起来像 run 没输出。
修法：把"focus/highlight message"和"truncate to message"拆成两个状态；Retry 只能 focus，不能进入截断模式。

`v1_thread_graph.go` (line 89) graph API 只授权入口 thread，后面整棵 fork tree 都按 `account_id` 拉。
风险：如果 fork family 内出现 owner/private/project 可见性漂移，用户可通过 graph 读到同 account 下其它 thread 消息。
修法：对 family 内每个 thread 做同样授权，或在 repo/DB 层强约束 fork family owner invariant，并补 mixed-owner/private 测试。

`threads_repo.go` (line 212) 递归 CTE 用 `UNION ALL`，没有 cycle/depth guard。
风险：脏数据或未来 bug 造成 `parent_thread_id` 自环/环形引用时，graph 请求可能打爆 DB 直到 timeout。
修法：CTE 加 visited path 或 CYCLE，再加最大 depth；DB 至少加 `id <> parent_thread_id` 检查。

**P2 应该尽快修**

`ConversationGraphPanel.tsx` (line 162) threadId 变了但旧 graph 不清空。
风险：切 thread 时右侧短暂显示上一条 thread 的 graph，而且还能点，可能跳回旧 thread 或制造过期 handoff。
修法：threadId 变化时清空 graph/禁用点击，或者响应回来时校验 request threadId。

`ChatView.tsx` (line 1915) 跨 thread 点图节点时先 `listMessages`，promise 返回后无条件 navigate。
风险：用户快速点多个节点或中途切 thread，旧请求返回后会抢导航。
修法：加 request sequence 或 `AbortController`，then 阶段确认目标仍是最新点击。

`messages_repo.go` (line 351) graph 全量读取整棵树所有消息，没有上限。
风险：大量 fork / 长 thread 会让右侧面板触发大 SQL、大 JSON 和大内存占用。
修法：定义 max threads/depth/messages；超过上限返回受控错误或窗口图。

`v1_thread_graph.go` (line 174) graph projection 用可见消息表找 `branched_from_message_id`。
风险：anchor 消息被 hidden/soft delete 后，branch merge 会断，图会重复前缀或变孤链。
修法：fork 时持久化 `branched_from_thread_seq`，或 graph 单独读取 anchor 的最小结构信息，不暴露内容。

`thread-list.tsx` (line 368) 后端列表隐藏 branch 后，前端全量 sync 会把本地 branch 运行态冲掉。
风险：branch run 正在跑时，重连/同步可能导致 active-session、未读完成、通知状态不稳定。
修法：展示层过滤 branch，但状态层保留已知 branch entries / running ids。

**P3 语义风险**

`threads_repo.go` (line 807) 搜索也过滤了 branch thread。
风险：branch-only 内容不可搜索，用户只能通过 root graph 找回。
修法：如果只是侧边栏不显示 branch，搜索不应该过滤；或者搜索 root 时返回 branch hit 信息。

**建议处理顺序**

先修 1、2、5、6，这几个直接影响你现在 dogfood 的稳定感。然后修 3、4、7、8，这几个是后端稳定性和权限边界。最后决定 9、10 的产品语义：branch 是否在"状态层可见但列表层隐藏"。

---

> 有没有减法修复，不然会变成补丁叠补丁式修复啊

有，而且应该用减法修。现在的问题不是"缺几个 guard"，而是这次把 graph 直接插进了现有 thread/run 生命周期，导致状态边界变脏。补丁式修会越补越散。

我建议的减法方向是：

**前端只保留一个事实状态：当前执行 thread**

Graph 里点击另一条 branch，不要搞 handoff、预取、半导航、半保留旧 UI。
直接切换当前 thread，让现有 ChatView 生命周期负责加载。
要减少刷新感，应该优化 thread 切换加载体验，而不是在 graph 里绕一层临时 handoff。

要删：
- `graphNavigationHandoffRef`
- `graphFocusMessageId` 作为跨 thread handoff 的用途
- 跨 thread 点击前的 `agentClient.listMessages(targetThreadId)`
- "旧图继续展示直到新图回来"的逻辑

**拆掉 `selectedGraphMessageId` 的双重语义**

现在它同时表示：
1. graph 里选中了某条消息
2. ChatView 只显示到某条消息
3. 发送时从这里 fork

这是核心混乱源。减法修法是拆成两个非常明确的状态：
- `graphSelectedMessageId`：只负责右侧图高亮
- `conversationCutoff` 或 `branchDraftAnchor`：只在用户明确"从这里继续写"时存在

更狠一点：先不要做"点击 graph 节点就截断左侧 ChatView"。点击 graph 节点只切 thread + scroll/focus。
只有点消息上的 branch 按钮，才进入"从这里继续发"的 anchor 模式。

**Retry 语义直接走 fork + navigate，不碰旧 thread run state**

Retry 创建 fork thread 和 run 后，不要在旧 ChatView 里 `setActiveRunId`。
只 `navigate('/t/forked', { state: { initialRunId } })`，让新 thread 自己接 run。
这样能删掉一批"旧 thread 消费新 run"的风险。

**Right panel 状态不按 branch thread 复制复杂缓存**

Conversation graph 是固定内置 tab，不应该像普通 resource tab 一样进入复杂 tab lifecycle。
可以保留：
- 是否打开 right panel
- 当前 active tab
- panel width

不要让 graph 跟每个 branch thread 的 saved right panel state 互相覆盖。
简单规则：如果 graph 打开，切 branch 后仍打开 graph；其它 resource tabs 按原逻辑恢复即可。

**后端 graph API 做成受限投影，不做全量世界观**

现在后端 graph 拉整棵 fork tree + 全量消息。减法修法：
- 明确只支持同 owner、同 account、未删除 thread
- CTE 加 depth/thread/message 上限
- 不支持的复杂情况直接返回受控错误，不做隐式兼容
- fork anchor 需要稳定字段，最好补 `branched_from_thread_seq`，否则 hidden anchor 迟早会断

**搜索不要顺手改语义**

"侧边栏不显示 branch"不等于"搜索不搜索 branch"。
这里应该回滚 search 的 `parent_thread_id IS NULL` 过滤。
展示列表过滤 branch 是 UI 需求，搜索是找内容，两者不是同一个语义。

我会把修复切成一个干净的 follow-up commit：
1. 删除 graph handoff 和预取导航
2. 拆掉 selected message 的混合语义
3. Retry 不再提前 `setActiveRunId`
4. graph panel thread 切换时清旧图或禁用旧图点击
5. 后端 CTE 加权限/环/上限基础保护
6. 回滚 search 隐藏 branch

这不是"再补几个 if"，而是把这次接入里最危险的隐式状态删掉。

---

> 2 这个想改的业务语义不太认同，我未来会加垂直列，那个时候单个 thread 的浏览不该靠 graph

认同，你这个反对是对的。第 2 点不能改成"graph 只负责高亮、不影响 ChatView"，那会把你想要的 FlowChat 语义砍掉。

我真正要减掉的不是业务语义，而是现在这个状态变量的混乱实现。

更准确的语义应该是：
- Graph 点击某条消息：ChatView 渲染到那条消息为止。
- 此时继续发送：从这个位置 fork 出新 thread。
- 单 thread 内更细的浏览，未来交给垂直列，不靠 graph。
- Graph 不是阅读器，是"分支定位器 / 执行上下文切换器"。

所以减法修法要改成这样：

保留"点击 graph 后 ChatView 截断"的语义。
但把状态命名和职责改正确，不叫 `selectedGraphMessageId` 这种 UI 状态。
用一个明确的领域状态，比如：

```typescript
type ConversationCursor =
  | { kind: 'tail'; threadId: string }
  | { kind: 'message'; threadId: string; messageId: string }
```

它表示"当前 ChatView 展示到哪里"，不是"图上选中了什么"。

然后规则清楚很多：
- `cursor.kind === 'tail'`：显示完整 thread，发送就是 append。
- `cursor.kind === 'message'`：显示到该 message，发送就是 fork。
- graph 高亮只是 cursor 的派生 UI。
- Retry 创建新分支后，cursor 应该回到新 thread 的 tail，而不是停在 anchor message。
- 跨 branch 点击只是在设置新的 cursor 和 active thread，不需要临时 handoff 状态。

这样不改变你的业务语义，反而把它固定下来。

所以我会修正刚才的减法方案：
1. 不取消 graph 控制 ChatView 截断。
2. 删除 `selectedGraphMessageId` 这种 UI 混合状态。
3. 改成 `conversationCursor` / `viewCursor`。
4. Retry、发送、graph 点击都只围绕这个 cursor 做状态转换。
5. 未来垂直列也可以直接消费这个 cursor，而不是再发明一套浏览状态。

这才是正确的架构，不是补丁。

---

> 同意，开修，依旧减法

按减法修完了，没继续叠 handoff / 预取 / 旧状态补丁。

这次改动核心：

- 把 `selectedGraphMessageId` 改成明确的 `ConversationCursor`
  - `tail`：显示完整 thread，发送 append
  - `message`：显示到某条消息，发送 fork
- 删除 graph 跨 thread 点击的 `listMessages` 预取和 `graphNavigationHandoffRef`
- 删除 `graphFocusMessageId`，改成语义更明确的 `cursorMessageId`
- Retry fork 后不再在旧 thread 上 `setActiveRunId`
  - 只通过 `initialRunId` 交给新 thread 的 ChatView 接管
- GraphPanel 切 thread 时不再展示旧图
  - 不是用 effect 清图，而是只渲染 `active_thread_id === threadId` 的 graph
- thread-list 同步时保留已知 branch thread 和 running branch ids，避免 root list 覆盖运行态
- 后端 graph API：
  - family 内每个 thread 都重新走读授权
  - 递归 CTE 从 `UNION ALL` 改 `UNION`，避免环形数据无限递归
  - 加 graph hard limit：200 threads / 2000 messages
- 搜索回滚 branch 过滤，侧边栏隐藏不再等于搜索隐藏

验证：
- `pnpm --filter web type-check` 通过
- `pnpm --filter web lint` 通过
- `go test ./internal/http/conversationapi ./internal/data` 通过

剩下没动的是 hidden/soft-deleted fork anchor 的长期稳定问题。那个最好不要补丁式猜，需要后续用 schema 正解：fork 时持久化 `branched_from_thread_seq`。当前这轮先把状态混乱和明显稳定性坑减掉了。

---

> commit

Commit: `89153e0b5` fix(web): stabilize conversation branch graph

## License

MIT
