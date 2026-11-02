# 008 食谱

这份食谱展示如何扩展 dsh-superagent。它使用 [001 领域模型](001-domain.md) 和 [002 授权](002-authorization.md) 中的术语，以及 [007 架构](007-architecture.md) 中的组件。

## 添加一个来源

这个配方实现一个名为 `webhook` 的来源种类。它通过一个虚构的 CLI `hookq` 读取一个假想的消息队列，并把消息转换成触发。所有其他接口都是 dsh-superagent 的一部分。

实现有四项职责：翻译事件、定义接收和授权规则、提供回复和上下文，以及维护轮询循环。编号步骤按顺序构建这些部分。

示例假定这个 CLI 契约：

- `hookq pull --after <n> --json` 打印位置 `n` 之后的消息，最旧在前：`{"me": "<the agent's id on the queue>", "items": […], "next": <position>}`。条目有 `id`、`conversation`、`thread`（启动其线程的消息的 id，在该消息上不存在）、带 `id` 和 `bot` 的 `sender`、`mentions_me`、`text`、`at` 和 `url`。
- `hookq reply --conversation <id> --to <message id> --key <key> --text <text>` 在消息下回复，每个键一次。对被删除的会话，它在标准错误上以 `{"error": {"code": 410, "message": "…"}}` 失败。
- `hookq thread --conversation <id> --root <message id> --json`，用 `--after <cursor>` 取更后一页，打印线程的一页，最旧在前：`{"items": […], "next": "<cursor>"}`，最后一页没有 `next`。

把种类及其插件入口放在 `src/sources/webhook.ts`，遵循 `src/sources/github.ts`。如果来源增长，把翻译、渠道和轮询代码拆成单独模块，如飞书来源那样。

### 1. 把事件映射到领域

在写代码前选择稳定的映射。触发 id、线程 id 和渠道地址是持久化的，所以以后更改它们可能断开现有记录。

| 概念 | 对 webhook 队列 |
| --- | --- |
| 触发 Trigger | 一条消息 |
| 线程 Thread | 一个会话中的一条消息及对它的回复：`<conversation>:<id of the first message>` |
| 参与者 Actor | 队列上发送者的 id，如允许名单所命名 |
| 渠道地址 | 回复所依附的会话和消息 |
| 接收范围 | 一个会话，在准入表中有自己的一行 |

### 2. 配置：schema、字段、事实和健康

模块导入以下步骤使用的内容：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { setTimeout as delay } from 'node:timers/promises'
import { z } from 'zod'
import { key, threadKey, type OutboundMessage, type Trigger } from '../core/index.ts'
import { CliError } from '../infra/commands.ts'
import { KV } from '../kv.ts'
import { health, sourceContext, type SourceContext } from './host.ts'
import type { SourceKind } from './kind.ts'
```

```ts
const ConfigSchema = z.object({
  conversations: z.array(z.object({
    id: z.string().regex(/^[a-z0-9-]+$/),
    listen: z.enum(['mentions', 'all']),
  }).strict()).default([]),
  pollSeconds: z.number().int().min(10).max(3600).default(30),
}).strict()
export type WebhookConfig = z.infer<typeof ConfigSchema>

export const webhookKind: SourceKind<WebhookConfig> = {
  kind: 'webhook',
  title: { zh: 'Webhook 队列', en: 'Webhook queue' },
  description: { zh: '接收 webhook 队列里下列对话的消息。', en: 'Receives the messages of these conversations from the webhook queue.' },
  schema: ConfigSchema,
  fields: [
    { key: 'conversations', type: 'rows', label: { zh: '接收的对话', en: 'Conversations' }, columns: [
      { key: 'id', type: 'text', label: { zh: '对话', en: 'Conversation' }, pattern: '[a-z0-9-]+' },
      { key: 'listen', type: 'select', label: { zh: '接收', en: 'Receive' }, options: [
        { value: 'mentions', label: { zh: '仅 @', en: '@ only' } }, { value: 'all', label: { zh: '全部消息', en: 'All' } }] },
    ] },
    { key: 'pollSeconds', type: 'number', advanced: true, min: 10, max: 3600,
      label: { zh: '轮询间隔 / 秒', en: 'Polling interval / seconds' } },
  ],
  health: ['webhook'],
  // find, scope, channel, threadContext, policy and environment follow in the next steps.
}
```

- `kind` 是稳定的来源 id。它出现在触发 `source`、线程键前缀、`source:webhook` 设置块和来源策略中。
- `schema` 校验已存储配置并提供默认值。运营者保存实例之前，registry 从那些默认值创建一个启用的实例。让默认值什么都不接收；这个例子默认没有会话。
- `fields` 无需自定义面板代码即可生成来源设置表单。`rows` 字段按第一列键控编辑条目；`advanced` 隐藏较少用的设置。如果来源能把 id 解析成显示名，实现 `refreshLabels`。把那些名字存到 `KV.sources.labels(kind)` 下。带 `choose` 和 `refresh` 的列随后可以提供它们。
- 可选的 `facts(deployment)` 显示设置派生的值，例如 CLI 背后的账号。设置目前只检查 `SETUP_FIELDS` 中列出的 GitHub 和飞书字段，所以这个例子不提供事实。
- `health` 声明来源的健康条目，初始 `starting`。注册认领每个名字，拒绝与 `runtime`、`reply`、`worker` 或 effect 种类之类的条目的冲突。用种类的名字，并把 `sync:<id>` 留给策略同步。`error` 会让状态脚本退出 2 并让每小时检查通知运营者，所以只在可能需要人的行动时才报告它。

### 3. 把消息翻译成触发

转换前先校验 CLI 响应：

```ts
/** One message as `hookq pull` and `hookq thread` print it. */
const MessageSchema = z.looseObject({
  id: z.string().min(1), conversation: z.string().min(1), thread: z.string().optional(),
  sender: z.looseObject({ id: z.string(), bot: z.boolean().default(false) }), mentions_me: z.boolean().default(false),
  text: z.string(), at: z.string(), url: z.string().optional(),
})
type QueueMessage = z.infer<typeof MessageSchema>
/** The answer of `hookq pull`, whose items are read one by one, and of `hookq thread`. */
const PageSchema = z.looseObject({ me: z.string(), items: z.array(z.unknown()), next: z.number().int() })
const ThreadPageSchema = z.looseObject({ items: z.array(MessageSchema), next: z.string().optional() })
```

```ts
/** The Trigger of one queue message; the agent's own messages produce none. */
export function webhookTrigger(message: QueueMessage, me: string): Trigger | undefined {
  if (message.sender.id === me) return undefined
  const thread = `${message.conversation}:${message.thread ?? message.id}`
  return {
    id: key('webhook', message.id),
    source: 'webhook',
    threads: [{ source: 'webhook', id: thread }],
    actor: { source: 'webhook', id: message.sender.id },
    human: !message.sender.bot,
    ...(message.mentions_me ? { addressed: 'mention' as const } : {}),
    at: message.at,
    text: message.text,
    channel: { source: 'webhook', thread: `webhook:${thread}`,
      address: { conversation: message.conversation, message: message.id, ...(message.url ? { url: message.url } : {}) } },
    ...(message.url ? { url: message.url } : {}),
    meta: { conversation: message.conversation, messageId: message.id, root: message.thread ?? message.id },
  }
}
```

- **稳定 id。** 通过 `key` 从队列 id 派生触发 id。`Engine.receive` 对已存储 id 返回 `duplicate`，重复接单只创建一个工单。因此重读消息是安全的。要把编辑当作新事件，在 id 中包含编辑版本，如 GitHub 用评论 `updated_at` 那样。
- **线程。** 第一个线程是主线程并拥有评估。额外线程添加关联上下文。来源选择 id；其完整键是 `webhook:<id>`。
- **发送者。** 在 `actor` 中使用来源原生的 id。对机器人和 CI 设 `human: false`。名单上的机器人只能成为推断候选。空的参与者 id（如 CI）使用“不在名单上、未针对智能体”发送者的准入格。只在消息确实针对智能体时把 `addressed` 设为 `mention` 或 `direct`；这让允许的人的消息有明确强度。绝不为智能体自己的消息产生触发。
- **回复目的地和元数据。** `channel` 标识回复去哪里；`url` 链接到来源消息。`channel.thread` 为有序投递分组消息：工单的后续消息排在它的更早消息之后，直到更早消息失败三次。其他工单不等。把来源特定字段放在 `meta` 中，例如接收会话和线程根。`meta` 和 `text` 都是不可信数据，不是授权。

智能体可以通过 `orchestrator_read trigger/<id>` 和 `thread/<key>` 读取新种类。这些暴露消息文本、去向和相邻触发。新种类不会自动获得内置技能、原始服务记录的资源或拉取请求绑定。

文本搜索无需额外代码即可工作。当人们使用来源特定的引用（例如消息 id、URL 或 `#12`）时，实现 `find(query)`。返回那些引用所标识的线程键和搜索词：

```ts
  find: query => ({ threads: [], terms: [...new Set(query.match(/\bm[0-9]+\b/g) ?? [])] }),
```

这里查询中的队列消息 id（例如 `m42`）会在触发中搜索它。

### 4. 来源接收什么：scope

```ts
  scope({ meta }, config) {
    const conversation = config.conversations.find(item => item.id === meta.conversation)
    if (!conversation) return { listened: false, reason: 'conversation_not_listed' }
    return { listened: true, observed: conversation.listen === 'all', row: conversation.id }
  },
```

Engine 通过 `SourceRegistry` 对每个传入触发调用 `scope`。范围外的消息连同其理由存储为已过滤，不创建线程。把消息传给 `receive`，而不是在轮询循环里静默丢弃它们，这样面板可以解释过滤。

`observed: true` 意味着来源在该范围接收未针对智能体的消息，允许对它们做准入表决定。`row` 标识对应的准入行。

重新校验用授权存储的元数据调用 `scope`。让它只依赖 `meta` 和配置。恢复要求 `listened`，不要求 `observed`。没有 `scope`，种类接收它产生的每个触发但不观察任何一个。

### 5. 回复：渠道

```ts
  channel: {
    allowed: (address, config) => config.conversations.some(item => item.id === address.conversation),
    send: sendWebhook,
    link: address => ({ url: address.url ?? '',
      label: { zh: `Webhook 对话 ${address.conversation}`, en: `Webhook conversation ${address.conversation}` } }),
  },
```

```ts
export async function sendWebhook(host: Pick<SourceContext, 'commands'>, message: OutboundMessage): Promise<'sent' | 'gone'> {
  const { conversation, message: to } = message.channel.address
  if (!conversation || !to) throw new Error('Missing webhook reply target')
  try {
    await host.commands.run('hookq', ['reply', '--conversation', conversation, '--to', to, '--key', message.id,
      '--text', message.text])
    return 'sent'
  } catch (error) {
    if (error instanceof CliError && error.code === 410) return 'gone'
    throw error
  }
}
```

- **幂等发送。** Outbox 调用 `SourceRegistry.send`，它调用 `channel.send`。消息在来源插件卸载时等待。发送可能在远程服务收到后失败，所以重试绝不能重复消息。用 `message.id` 作为幂等键：`hookq` 支持 `--key`；GitHub 存隐藏评论标记；飞书接受幂等键。
- **永久与可重试失败。** 成功返回 `sent`，只在投递永久不可能时返回 `gone`，例如例子的 410。gone 消息保留为已丢弃且绝不再次排队；它出现在 `GET sources` 和来源页上。其他失败抛错，让 Outbox 在 5 秒后重试，翻倍到 300 秒。`CliError.code` 是 CLI 输出报告的代码。
- **允许的目的地。** `allowed(address, config)` 只检查已保存地址和当前配置。把每个必需事实放在地址里；例如，飞书记录聊天是否为私聊。不再被监听的目的地被丢弃，而被禁用的来源让消息等待。没有 `allowed`，来源启用时允许所有地址。
- **链接。** `link(address, links)` 返回面板通过 `SourceRegistry.link` 显示的 URL 和标签。

### 6. 给研判的线程上下文

```ts
  threadContext: {
    messageId: trigger => String(trigger.meta.messageId ?? ''),
    // A queue thread is flat: its root, and the message a reply answers, are on its first page, as on GitHub.
    read: async (commands, trigger, _part, cursor) => {
      const page = ThreadPageSchema.parse(JSON.parse(await commands.run('hookq', ['thread', '--conversation',
        String(trigger.meta.conversation), '--root', String(trigger.meta.root), '--json', ...(cursor ? ['--after', cursor] : [])])))
      const messages = page.items.map(item => ({ id: item.id, from: item.sender.id, at: item.at, text: item.text,
        ...(item.sender.bot ? { bot: true as const } : {}), ...(item.url ? { url: item.url } : {}) }))
      return { messages, ...(page.next ? { next: page.next } : {}), body: { messages } }
    },
  },
```

主动研判需要触发周围的会话。`orchestrator_eval` 读初始窗口；`orchestrator_eval_read` 展开页、根或父消息。实现可选的 `threadContext` 接口来提供这些读取：

- `read(commands, trigger, part, cursor, config)` 返回 `ContextPart`。其 `messages` 使用 `ContextMessage`：`{id, from, at, text}` 加适用的 `bot`、`deleted`、`reply_to`、`url`、`path` 和 `line` 字段。最后一页省略 `next`。`body` 是展示给研判 Session 的内容，这里就是同样的消息。可选的 `status` 可以描述特殊结果，例如飞书的 `chat_neighbours`。`part` 是 `thread`、`root` 或 `parent`。
- `messageId(trigger)` 在其中找到触发自己的消息。
- `images(text)` 和 `image(…)` 用于其消息带图片的种类，如飞书的；例子省略它们。

没有 `threadContext` 的种类给研判的没有线程上下文；触发自己的文本仍可读。

如果消息回复更早的消息，添加 `repliesTo(trigger)`。它返回对应的触发 id，允许后续通知和 worker 输入在存储后显示被回复的消息。例如，如果 `hookq` 提供 `reply_to` 且翻译把它存为 `meta.replyTo`：

```ts
  repliesTo: trigger => typeof trigger.meta.replyTo === 'string' ? key('webhook', trigger.meta.replyTo) : undefined,
```

### 7. 谁能请求：来源的策略部分

```ts
  policy: {
    schema: z.object({ allowed: z.array(z.object({ id: z.string().min(1), name: z.string().trim().max(100).optional() }).strict())
      .default([]) }).strict(),
    fields: [{ key: 'allowed', type: 'rows', label: { zh: '能发起工作的发送者', en: 'Senders who may originate work' }, columns: [
      { key: 'id', type: 'text', label: { zh: '发送者', en: 'Sender' } },
      { key: 'name', type: 'text', label: { zh: '名字', en: 'Name' }, optional: true }] }],
    explicit: { zh: '名单里的人在消息里 @ 智能体。', en: 'Someone on the list mentions the agent in a message.' },
    rows: stored => ConfigSchema.parse(stored).conversations.map(conversation => {
      const all = conversation.listen === 'all'
      return { id: conversation.id, label: conversation.id, note: all ? { zh: '全部消息', en: 'All' } : { zh: '仅 @', en: '@ only' },
        cells: { listedUnaddressed: all, unlistedAddressed: true, unlistedUnaddressed: all } }
    }),
  },
```

- `schema` 校验来源策略，包括 `allowed` 发送者 id。id 比较不区分大小写。准入表单独校验并在种类间共享。这个策略部分保存在 `policy.sources.webhook` 并贡献给策略版本。
- `fields` 在 **Eval → On each Source** 下生成表单，以 `allowed` 开头。`explicit` 解释如何针对智能体。可选的 `unnamed: {person, bot}` 为不透明的 id（例如飞书的 `ou_…`）提供显示标签；这个例子省略它，因为队列 id 是可读的名字。
- `rows(config)` 列出准入范围及其可用格。每行 id 必须匹配 `scope` 返回的 `row`。未针对智能体的消息的格只在来源观察那些消息的地方可用。没有 `rows` 的种类使用默认格。
- 可选的 `bots` 列出有资格成为推断候选的机器人，使用与 `allowed` 相同的条目形状。没有它，没有机器人被准入。
- 可选的 `sync: {label, ready(part), run(host, part, signal)}` 在启动时和 **Sync now** 时刷新成员关系。飞书使用群成员关系。失败出现在 `sync:<id>` 下并在一分钟后重试，翻倍到一小时直到成功。

### 8. 给执行器的变量

```ts
  environment: {
    variables: [{ key: 'DSH_SUPERAGENT_WEBHOOK_CONVERSATION', description: {
      zh: '接下工单的 webhook 对话；工单不是从这个来源来的时为空',
      en: 'The webhook conversation that started the ticket; empty when the ticket did not start on this Source' } }],
    values: ({ trigger }) => ({
      DSH_SUPERAGENT_WEBHOOK_CONVERSATION: trigger?.source === 'webhook' ? String(trigger.meta.conversation ?? '') : '' }),
  },
```

执行设置在 prepare 命令旁显示声明的 `variables`，即使来源被禁用。启用时，`values` 使用部署、配置和起始触发提供工单特定的值。

prepare 命令读环境变量；worker 在其指令中读它们的值。使用 `DSH_SUPERAGENT_<KIND>_…` 前缀，对来自另一来源的工单返回空值，绝不包含凭据。

### 9. 插件入口及其循环

```ts
export const name = 'dsh-superagent-webhook'
export const inject = ['orchestrator']
export function apply(ctx: Context): void {
  ctx.effect(() => ctx.orchestrator.sources.register(webhookKind), 'webhook: source kind')
  ctx.effect(() => {
    const source = sourceContext(ctx.orchestrator, webhookKind)
    const controller = new AbortController()
    // DSH ends its process on an unhandled rejection, so a failure that escapes the loop becomes the Source's health.
    const running = watch(source, controller.signal)
      .catch((error: unknown) => { source.report('webhook', { state: 'error', detail: String(error) }) })
    return async () => { controller.abort(); await running }
  })
}
```

插件注入 `orchestrator`。其第一个 effect 注册种类并在处置时注销它。第二个用 `SourceContext` 运行轮询循环。处置中止循环并等待完成，包括应用停止或重启期间。

```ts
async function watch(ctx: SourceContext, signal: AbortSignal): Promise<void> {
  await ctx.ready
  // Setup restarts dsh-superagent once it resolves the accounts; until then the Source reads nothing.
  if (!ctx.configured()) { ctx.report('webhook', health('paused', 'setupPending')); return }
  let failures = 0
  while (!signal.aborted) {
    const config = ctx.config<WebhookConfig>('webhook')
    if (!config) ctx.report('webhook', health('paused', 'sourceOff'))
    else {
      const at = new Date().toISOString()
      try {
        const { read, unreadable } = await pollQueue(ctx, signal)
        failures = 0
        ctx.report('webhook', { state: 'ready', detail: `${read} messages, ${unreadable} unreadable`, at })
      } catch (error) {
        failures += 1
        // One network failure is retried on the next round; three in a row need a person.
        if (!signal.aborted && failures >= 3) {
          ctx.report('webhook', health('error', 'pollFailures', { count: String(failures), error: String(error) }))
        }
      }
    }
    await delay((config?.pollSeconds ?? 30) * 1000, undefined, { signal }).catch(() => {})
  }
}

/** Read the queue after the cursor, take in each message, then move the cursor past them. */
export async function pollQueue(ctx: SourceContext, signal: AbortSignal): Promise<{ read: number; unreadable: number }> {
  const after = ctx.state.get(KV.webhook.cursor) ?? 0
  const output = await ctx.commands.withSignal(signal).run('hookq', ['pull', '--after', String(after), '--json'])
  const page = PageSchema.parse(JSON.parse(output))
  let unreadable = 0
  for (const item of page.items) {
    if (signal.aborted) throw new Error('The webhook Source stopped during the read')
    const message = MessageSchema.safeParse(item)
    if (!message.success) { unreadable++; continue }
    const trigger = webhookTrigger(message.data, page.me)
    if (!trigger) continue
    await ctx.receive(trigger)
    const conversation = { label: { zh: '对话', en: 'Conversation' }, value: message.data.conversation }
    for (const ref of trigger.threads) ctx.threads.describe(threadKey(ref), { facts: { conversation } })
  }
  ctx.state.set(KV.webhook.cursor, page.next)
  return { read: page.items.length, unreadable }
}
```

- **设置和开关。** 等待 `ctx.ready`，且 `ctx.configured()` 为 false 时什么都不读。每轮读 `ctx.config(kind)`：来源关闭时它未定义。
- **游标。** 用 `KV`（`src/kv.ts`）中的类型化条目把读取位置存到 `ctx.state`，例如 `webhook: {cursor: record<number>('webhook-cursor')}`。`tests/store.spec.ts` 要求键字面量只出现在该 registry 中。外部包可以用导出的 `record` 和 `family` 定义键。设 `undefined` 移除记录；`ctx.atomic` 在事务中分组写入。

  只在每个覆盖的触发都已存储后才推进游标。被中断的轮次让游标不变；稳定 id 使重放安全。对时间戳游标，`src/sources/poll.ts` 的 `cursorAt(started)` 减去五秒以容纳迟到的远程写入。
- **线程元数据。** 接收触发后，调用 `ctx.threads.describe(key, {label, url, facts})` 填充其显示信息。`ctx.threads.patch(key, meta)` 委托给 `threads.patchMeta`，设置来源拥有的键，或当值为 `undefined` 时移除它们。二者都忽略不存在的线程，包括已过滤触发的线程。
- **逐条错误。** 计数畸形条目并继续，而不是让整批失败。`src/infra/commands.ts` 的 `validItems(item, list, tally)` 支持这种模式。对独立轮询的线程，用 `src/sources/poll.ts` 的 `pollThreads`；它把每次失败记录为 `pollError`。其 `polled` 谓词保留活跃订阅，以及最后一个触发与工单结束二者中较晚者之后的两周窗口。
- **健康。** 只通过 `ctx.report` 写已声明条目。`health(state, code, values)` 提供固定消息，例如 `setupPending`、`sourceOff` 和 `pollFailures`。报告 `error` 前先数连续失败。
- **中止。** 把信号传给 `ctx.commands.withSignal(signal)` 和 `delay`，并在条目之间检查它，让停止或重启立即结束循环。

这个例子不发送来源发起的通知。需要它们的来源调用 `ctx.notify`；Outbox 像其他回复一样投递它们。

它也不把线程绑定到工单完成。只有当来源的线程生命周期决定工作是否完成时（如 GitHub 拉取请求那样），才用 `ctx.tickets.bindThread` 和 `threadClosed`。Engine 仍决定工单何时结束。

### 10. 注册插件

在 `tsdown.config.ts` 中把模块构建为它自己的入口：

```ts
entry: { index: 'src/index.ts', 'github-watch': 'src/sources/github.ts', 'feishu-watch': 'src/sources/feishu.ts',
  'webhook-watch': 'src/sources/webhook.ts' },
```

然后在 `package.json` 中通过把 `"./webhook-watch": "./lib/webhook-watch.js"` 加到 `exports` 来导出构建文件，并把插件加到 bundle 的 `cordis.patch.yml` 的 `insert` 列表，禁用，如飞书那样：

```yaml
    - id: dsh-superagent-webhook
      name: dsh-superagent/webhook-watch
      disabled: true
```

在配置文件的 `cordis.patch.yml` 中用 `id: dsh-superagent-webhook` 和 `disabled: false` 启用插件。构建并重启。新来源出现在 Sources 设置和 **Eval → On each Source** 下。

把它的模块和 spec 加到 [007 架构](007-architecture.md) 的模块和测试表中。`tests/docs.spec.ts` 检查两个索引都完整。

### 11. 测试

用 `tests/sources.spec.ts` 的模式创建 `tests/webhook.spec.ts`：临时目录中的真实 Store、带 `HealthBoard` 的 `SourceRegistry`、带替身端口的 Engine，以及来自 `sourceContextOf` 的 `SourceContext`。

通过侦听 `Commands.run` 模拟 CLI 调用。也侦听 `withSignal`，因为它通常返回新的 `Commands` 实例。

```ts
function host(overrides: Partial<SourceContextDeps> = {}) {
  const store = new Store(join(mkdtempSync(join(tmpdir(), 'webhook-')), 'db.sqlite'))
  store.saveSettings(SettingsSchema.parse({
    sources: { webhook: { enabled: true, config: { conversations: [{ id: 'ops', listen: 'mentions' }] } } },
    policy: { sources: { webhook: { allowed: [{ id: 'alice' }] } } },
  }))
  // The registry claims the kind's health entries on the board, where the test reads them.
  const board = new HealthBoard()
  const sources = new SourceRegistry(() => store.settings(), board)
  sources.register(webhookKind)
  const engine = new Engine({ repo: store, scope: sources, policy: () => settingsPolicy(store.settings(), sources),
    assessor: { request: vi.fn() }, executor: { start: vi.fn(), deliver: vi.fn(), stop: vi.fn() },
    outbox: { send: vi.fn(), status: () => undefined, withdraw: vi.fn() } })
  const commands = new Commands(() => testDeployment(), {}, new AbortController().signal)
  const receive = vi.fn((trigger: Trigger) => { const result = engine.receive(trigger); engine.evaluateDue(); return result })
  // The Source's own notices go to the outbox, which keeps their receipts and sends nothing here.
  const deps: SourceContextDeps = { store, engine, outbox: { send: message => { store.receipts.add(message) } }, commands, sources,
    deployment: testDeployment(), ready: Promise.resolve(), configured: () => true, receive, ...overrides }
  return { ...deps, store, engine, commands, receive, health: board.all, ctx: sourceContextOf(deps, webhookKind) }
}
/** The queue as `hookq` answers it. */
function queue(h: ReturnType<typeof host>, answer: () => unknown) {
  vi.spyOn(h.commands, 'withSignal').mockReturnValue(h.commands)
  return vi.spyOn(h.commands, 'run').mockImplementation(async () => JSON.stringify(answer()))
}

it('takes in what the queue holds after the cursor, once, and moves the cursor after it', async () => {
  const h = host()
  queue(h, () => ({ me: 'agent', items: [message('m1'), { id: 'broken' }, message('m2', { conversation: 'elsewhere' })], next: 3 }))
  expect(await pollQueue(h.ctx, new AbortController().signal)).toEqual({ read: 3, unreadable: 1 })
  expect(await h.receive.mock.results[0]!.value).toBe('stored')
  expect(await h.receive.mock.results[1]!.value).toBe('filtered')
  expect(h.store.get(KV.webhook.cursor)).toBe(3)
  // An allowed sender who mentions the agent makes an explicit request.
  expect(h.engine.candidate(key('webhook', 'm1'))?.strength).toBe('explicit')
  await pollQueue(h.ctx, new AbortController().signal)
  expect(await h.receive.mock.results[2]!.value).toBe('duplicate')
  h.store.close()
})
```

`message(id, fields)` 构建队列条目。至少覆盖：

- 翻译：稳定 id、线程、参与者、`human` 和 `addressed`，以及智能体自己的消息无触发；
- 一轮：触发存储一次、范围外的一条 `filtered`、游标在其后移动、允许发送者的提及被评估为 `explicit`；
- 被截断的一轮，保留其游标；
- `send`：`sent`、仅对 410 的 `gone`，以及任何其他失败的抛错，用 `commandError` 构建 CLI 错误；
- 线程上下文：一页的消息和下一页的游标，如 `tests/registry.spec.ts` 通过 `SourceRegistry.threadContext` 读取种类的线程那样；
- 插件：`apply` 注册种类，设置等待时报告 `setupPending`，其处置器注销种类，逃出循环的失败成为来源的健康而非未处理的拒绝；
- 来源的策略部分及其准入行，如 `tests/registry.spec.ts` 和 `tests/policy.spec.ts` 检查飞书来源的那样。

### 清单

1. 映射触发、线程、参与者、渠道地址和接收范围。
2. 写种类：`schema`、`fields`、`health`，以及如果设置派生了什么则 `facts`。
3. 用稳定 id 把事件翻译成触发，并丢弃智能体自己的。
4. 在 `scope` 中只从 `meta` 决定来源接收什么。
5. 给它一个渠道，其 `send` 每条消息发一次，只在会话永远无法接收它时回答 `gone`。
6. 如果它能读线程，给研判其线程上下文。
7. 给它带 `allowed` 和准入行的策略部分。
8. 注册其执行环境变量。
9. 写插件入口和一个循环，它等待设置、跟随开关、在 `receive` 后移动游标、逐条保留错误、报告健康并在中止时停止；在 `KV` 中命名游标。
10. 添加构建入口、导出和 `cordis.patch.yml` 插入，并在 007 的表中列出模块及其 spec。
11. 如 `tests/sources.spec.ts` 测试其他来源那样测试它。
