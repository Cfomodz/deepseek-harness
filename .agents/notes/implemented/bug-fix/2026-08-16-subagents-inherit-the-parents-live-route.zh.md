# Agent Note: Subagents inherit the parent's live route

Status: implemented

[English](2026-08-16-subagents-inherit-the-parents-live-route.md) | 中文

## 问题

harness 里存在两种"这个 agent 用哪个模型"的表示。`Agent.options` 是 `agents.create()` / `agents.resume()` 时声明的种子。真正生效的路由是 `agent/request` waterfall 的结果——它既是实际派发出去的配置，也是 loop 追加为 `request/header` 的值，随时可通过 `session.requestHeader()` 读取。

所有在运行期改变路由的机制都作用于后者。`installModelSelection()` 替换已解析请求配置上的 provider、model 与 effort，从不写回 options；api-proxy 的选择分层在每次读取时重新解析：进程内选择、该会话自己记录的 header、部署默认值。在 Web 宿主下，`agentOptions()` 为每一次 create、resume 与 fork 都从 `defaults.defaultModelSelection()` 播种，因此一个会话的 options 记录的是它被创建那一刻的部署默认值，此后不再变化。

`resolveChildAgentOptions()` 读的是 options。二者仅在"刚创建、部署默认值未变、第一次请求"时相等，于是一个在选择器里切换过模型的父 agent，会把子 agent 派发到它被创建时的那条路由上，而它自己的轮次与自己的日志都正确显示着切换后的路由。

provider id 不是显示偏好：它选定已注册的 adapter，而每个 adapter 自带凭据引用与 base URL。继承了陈旧 provider 的子 agent 会用另一把 key、向另一个端点认证，被计费的是那条陈旧路由对应的账户——这正是 `dsh-llm-pi-ai` 在拒绝回退到环境中的 key 时已经写明的后果："billing another tenant for a request the deployment meant to authenticate differently"。而且没有任何东西暴露这个分歧：父 agent 自己的 header 全程正确，子 agent 上没有报错、没有警告、也没有任何 UI 信号。

持久化 descriptor 让它跨越了进程生命周期。`startContinuable()` 从同一份陈旧 options 快照出 `agentProvider` / `agentModel`，冷恢复据此重建子 agent 的 `agentOptions`，孙 agent 又从子 agent 那份已经错误的 options 继承。父 agent 的一次错误轮次，会跨重启污染整棵委派子树。

同一个文件里，仓库已经就这个问题做过两次决策，两次都选择了父 agent 的活动状态。`childSessionMeta()` 从父 agent 的活动 scope 链而非其 header 读取 preset，"因为一个在空白状态下切换过 preset 的父 agent 运行在更新的组合上，而它的 header 仍然写着旧的"。`captureDelegatedPolicyOverrides()` 读取父会话显式的活动沙箱覆盖值，并且刻意不读部署默认值。唯独模型路由读的是创建快照，既没有 Agent Note 也没有成文规则——它多半是照着 `Agent.options` 的 JSDoc 写出来的，那句话声称自己是"this agent's requests use"的 provider 路由与模型，而自请求级选择上线之后它就不再成立了。

## 决策

子 agent 继承父 agent 最近一次请求实际使用的路由：provider 与 model 取自 `session.requestHeader()?.config`，只有当父 agent 从未发出过请求时才回退到 `parent.options`，而显式的每次调用 `agentOptions` 始终位于最外层。`dsh-subagent` 的 `child-agent.ts` 中的 `parentRouteOf()` 就是这唯一一处解析，因此两个进程内驱动都从拥有委派契约的那个包获得规则，而不是由每个 Consumer 各自复述一遍。

adapter 提供的 `maxTokens` 不被继承。`LlmCallConfigAdapterDefaults.maxTokens` 标记的是"调用方未提供、由精确模型的 adapter 解析出来"的上限；把它提升为子 agent 的显式选项，会让该子 agent 在此后运行的每一个模型上都被钉死在某一个 adapter 的默认值上。显式的父 agent 上限永远不会被标记，因此会正常继承。这与 `requestProposal()` 的做法一致——后者在插件提出下一次配置之前，剥离的正是这些被标记的字段。

`startContinuable()` 现在在第一个 await 之前一次性解析子 agent 的选项，并由这一个值同时导出子 agent 的 `AgentOptions` 与持久化 descriptor 的 `agentProvider` / `agentModel`。在 await 之前解析，与紧随其后的委派策略捕获保持一致：路由属于委派发生那一刻的父 agent，而不是 materialization 拿到锁的那一刻。由同一个值导出两者，消除了 descriptor 此前复述的那份重复优先级规则，并且无需改动恢复路径就修好了冷恢复——descriptor 里存的是正确路由，重建出来的就是正确选项。

`Agent.options` 现在被文档化为创建期种子，并指明 `session.requestHeader()?.config` 才是当前生效路由的权威来源。那句过时的 JSDoc 是本缺陷最可能的近因，若不修正，下一个消费者会再犯一次。

## 备选方案

**在选择器改变路由时，把选中的路由写回 `Agent.options`。** 否决，因为这与既定方向相反：请求级选择被刻意设计为作用于请求的 waterfall，好让并发切换在下一步生效，而不是在轮次中途改变 agent 状态；api-proxy 的分层也已经把日志当作权威。让 options 可变，等于给一个被当作创建期不变量来读取的值增加了第二个写者。

**恢复时从会话自己记录的 header 而非部署默认值播种 `agent.options`。** 未被否决——这是有意推迟，且在本次改动落地后与之正交。它会让恢复后的会话的 `agent.options` 变得诚实，也会与同一段 api-proxy 代码中它上方三行的 preset 解析保持一致（后者已经从会话日志解析）。但它并不能修复 Case A——那里根本没有发生恢复——而且委派现在也不再依赖它。将其留作独立决策，好让这个扣费缺陷单独落地。

**在错误种子的源头，也就是 api-proxy 里修。** 以角色错误为由否决。`resolveChildAgentOptions()` 正是委派 seam 所拥有的 `resolve(request): Spec` 步骤；改 Consumer 会让另一个进程内驱动仍然是错的，也会把路由规则放进宿主里。

**provider 与 model 各自独立取值、各自独立回退。** 否决，因为 provider 与 model 是一对——活动的 provider 配上快照里的 model，指向的可能是新 adapter 根本不提供的模型。已记录的 header 必定同时携带两者，所以要么整对继承，要么都不继承。

## 测试

`packages/subagent/subagent-in-process-driver/tests/subagent-in-process-driver.spec.ts` 在一个把路由切换到第二个已注册 adapter 的 `agent/request` 监听器下驱动父 agent 跑完一轮，然后断言：子 agent 的选项、以及子 agent 自己记录的 header，都指向切换后的路由；两次请求都到达了切换后的 adapter；创建期种子对应的 adapter 一次都没有被调用——这正是一个部署用来回答"这笔钱记到哪个账上"的审计路径。另外两个用例守护的是新的解析逻辑而非旧缺陷：显式请求路由仍然覆盖父 agent 的活动路由；adapter 解析出的 `maxTokens` 不被提升，而显式的父 agent 上限会被继承。既有的"无路由父 agent"用例继续钉住：对于完全没有路由的父 agent，不凭空捏造任何值。

`packages/subagent/subagent/tests/continuation.spec.ts` 断言持久化的 `subagent/descriptor` 记录的是父 agent 的活动路由，而这正是冷恢复重建时所依据的内容。既有的冷恢复用例继续钉住：显式请求的子 agent 路由能跨重启存活。

两处回归断言都经过验证：在旧解析器下失败，在新解析器下通过，因此它们钉住的是这次修复本身，而不是周边机制。

组装后转录这一层采用的是已发布 Web 组合的 e2e，而不是无密钥快照，理由与子 agent preset 继承落地时记录的一致：本仓库发布的可运行示例中，没有任何一个同时组合了模型选择器与委派，因此该缺陷在快照工具链里根本不可观测。要做快照场景，得先有这样一个示例。Web e2e 启动的是选择器与委派共存的真实组合，这是当下可得的组装证据。

## 影响

一次委派现在会读取父 agent 折叠后的请求 header，这是 session 本就为 loop 维护的、代价为 O(新事件数) 的增量读取。没有新事件、没有新服务方法、没有持久化变更。

子 agent 的路由跟随父 agent 的选择器，这既是用户切换模型时的预期，也让委派决策可以从日志中重建。此前依赖"子 agent 钉在启动默认值上"的部署，现在必须通过委派工具的 `agentOptions` 显式表达——那一直是同一个旋钮，只是现在成了表达该意图的唯一方式。

已存在的可继续子 agent 仍然保留冻结在其 descriptor 中的路由。本次修复纠正的是新委派所记录的内容，不会重写已经写入的 descriptor，因此在此改动之前创建的子 agent，冷恢复时仍会回到它创建时的那条路由。这与 descriptor 一贯的不可变性一致；给已存在的子 agent 改路由属于迁移，而不是修复。

在 Web 宿主下，恢复后的会话的 `agent.options` 仍然错报路由。现在委派不再读它，但未来若有消费者把它当作"当前路由"来读，就会重蹈覆辙——这正是修正后的 JSDoc 所要防止的，也正是上面那个推迟的备选方案将彻底消除的。
