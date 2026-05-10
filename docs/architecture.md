# BossHelper 架构说明

本文面向需要修改、扩展或排查问题的开发者,描述这个浏览器扩展的运行时拓扑、核心抽象与关键设计决策。配合 `CLAUDE.md` 的速记一起读。

## 目录

- [一、它要解决什么问题](#一它要解决什么问题)
- [二、三层运行时](#二三层运行时)
- [三、跨层 RPC:`comctx` 的桥接模型](#三跨层-rpccomctx-的桥接模型)
- [四、寄生 Boss 的 Vue 应用](#四寄生-boss-的-vue-应用)
- [五、投递流水线 Pipeline](#五投递流水线-pipeline)
- [六、AI 双轨:OpenAI 兼容 vs 服务端代理](#六ai-双轨openai-兼容-vs-服务端代理)
- [七、消息发送:寄生 Boss 的 WebSocket](#七消息发送寄生-boss-的-websocket)
- [八、缓存:LRU + 处理器分级 TTL](#八缓存lru--处理器分级-ttl)
- [九、账号切换:cookies + 配置 + 统计三位一体](#九账号切换cookies--配置--统计三位一体)
- [十、工程约定与陷阱](#十工程约定与陷阱)

---

## 一、它要解决什么问题

Boss 直聘的真实痛点不是"投简历",而是**「在海量岗位里识别真正值得投的、并且尽可能像人一样自然地打第一个招呼」**。要做到这点,有 4 个绕不开的硬骨头:

1. **页面的鲜活数据在 Vue 实例里**——HTML 是渲染后的快照,真正的 `jobList`、`pageVo`、`clickJobCardAction` 都挂在 Vue 组件上;
2. **要能改 Boss 自己的运行时**——比如劫持 `setter` 监听 `jobList` 变化、复用页面自己的 `pageChangeAction` 翻页;
3. **跨域请求与 cookie 切换**——浏览器扩展环境才能突破 CORS、操纵 `chrome.cookies`;
4. **AI 与真实聊天通道**——既要调外部 LLM,又要把聊天消息按 Boss 的 protobuf 协议塞进 Boss 自己的 WebSocket。

整个项目的架构,本质上就是为这 4 件事搭了 4 条管道。

---

## 二、三层运行时

这是理解一切的钥匙。扩展运行在三个隔离上下文中:

```
┌──────────────────────────────────────────────────────────────────┐
│  Browser Tab (zhipin.com)                                        │
│                                                                   │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐ │
│  │   MAIN WORLD            │    │   ISOLATED WORLD             │ │
│  │   (页面主世界)            │    │   (扩展隔离世界)               │ │
│  │                         │    │                              │ │
│  │  Boss 自己的 Vue          │    │  content.ts                 │ │
│  │  window.GeekChatCore    │    │   - injectScript('main-     │ │
│  │  window.Cookie          │    │     world.js')               │ │
│  │                         │    │   - ProvideContentAdapter    │ │
│  │  main-world.ts          │    │     (window.postMessage)     │ │
│  │   - 我们的 Vue 应用       │◀──▶│   - InjectBackgroundAdapter │ │
│  │   - getRootVue()        │ msg│     (runtime.sendMessage)    │ │
│  │   - injectCounter() ────┼────┼──▶ ContentCounter            │ │
│  └─────────────────────────┘    └──────────────┬───────────────┘ │
└──────────────────────────────────────────────────┼───────────────┘
                                                   │ runtime msg
                                                   ▼
                              ┌─────────────────────────────────┐
                              │  SERVICE WORKER (background)    │
                              │   - BackgroundCounter           │
                              │   - browser.cookies / fetch /   │
                              │     notifications               │
                              └─────────────────────────────────┘
```

**为什么必须三层?**

| 层         | 入口文件                        | 必须存在的理由                                                                                  |
| ---------- | ------------------------------- | ----------------------------------------------------------------------------------------------- |
| main-world | `src/entrypoints/main-world.ts` | 直接访问 Boss 的 Vue 实例(`#wrap.__vue__`)、`window.GeekChatCore` 这些主世界全局,隔离世界看不见 |
| content    | `src/entrypoints/content.ts`    | 唯一一个**既能用 `chrome.runtime` 又能操作 DOM** 的层,负责注入 main-world 脚本和做消息桥        |
| background | `src/entrypoints/background.ts` | Service Worker,独占 `browser.cookies` / `browser.notifications` / 跨域 `fetch` 这些扩展 API     |

main-world 脚本通过 `<script>` 标签注入(`web_accessible_resources` 配置在 `wxt.config.ts`),不是常规的 content script——所以它的 CSS 注入需要手工 hook 到 manifest(见 wxt.config.ts 的 `build:manifestGenerated`)。

---

## 三、跨层 RPC:`comctx` 的桥接模型

`src/message/` 三个文件用 `comctx` 库的 `defineProxy` 模式,把"跨上下文调用"伪装成普通方法调用。

```
main-world                  content                      background
─────────────               ────────────                  ─────────────
counter.cookieSwitch(uid)
   │
   ▼ window.postMessage              ProvideContentAdapter
                          ─────▶  ContentCounter.cookieSwitch
                                         │
                                         ▼ runtime.sendMessage
                                                              ─────▶ BackgroundCounter.cookieSwitch
                                                                       │
                                                                       ▼ browser.cookies.* (实际执行)
```

**namespace**:`__boss-helper-content__` 与 `__boss-helper-background__`,各自独立通道,避免和别的扩展或 Boss 自己的 `postMessage` 串台。

**ContentCounter 的角色**:几乎所有 `cookie*/request/notify` 方法都只是一行透传(`return this.background.xxx(...args)`)。它的存在不是为了逻辑,而是**给 main-world 一个语义统一的代理对象**——main-world 调 `counter.xxx()` 时,完全不用关心方法跑在哪一层。

ContentCounter 同时叠加了几个**只属于 content 层的能力**:

- `storageGet/Set/Rm` — `chrome.storage` API 在 content 层就能用,放在中间层效率最高
- `genKey()` 默认前缀 `sync:`,要存 `local:` / `session:` 必须显式带前缀

**新增跨层方法的标准流程**:

1. 在 `BackgroundCounter` 上加方法
2. 在 `ContentCounter` 中加一个透传方法(`return this.background.xxx(...args)`)
3. 在 main-world 调用 `counter.xxx()`,`comctx` 的 `defineProxy` 会处理类型推导

---

## 四、寄生 Boss 的 Vue 应用

这是项目最巧妙的部分。Boss 用的是 Vue 2,Vue 2 会把组件实例挂在 DOM 的 `__vue__` 属性上。`composables/useVue.ts` 提供三个工具:

### `getRootVue()`:拿到 Boss 的根

```ts
const wrap = document.querySelector('#wrap')
if ('__vue__' in wrap) rootVue.value = wrap.__vue__
```

之后能拿到 `$router`、`$store.state.userInfo` 等等。`main-world.ts` 通过 `v.$router.afterHooks.push(main)` 让自己跟随 SPA 路由切换。

### `useHookVueData()`:监听数据变化(setter 劫持)

```ts
const originalSet = jobVue.__lookupSetter__(key)
Object.defineProperty(jobVue, key, {
  set(val) {
    data.value = val // 同步到我们的 Pinia ref
    update?.(val) // 触发投递逻辑
    originalSet.call(this, val) // 不能阻断,要让 Boss 自己继续工作
  },
})
```

**没有**轮询、没有 `MutationObserver`,而是直接劫持 `jobList` 的 setter。每当 Boss 自己 fetch 来一页岗位赋值给 `jobList`,我们这边就同步拿到完整对象数组(已经包含 `encryptJobId`、`securityId` 这些后端字段)。

### `useHookVueFn()`:复用 Boss 自己的方法

```ts
pageChange.value = await initChange() // Boss 的 pageChangeAction
clickJobCardAction = await hookClickJobCardAction() // Boss 的卡片点击方法
```

翻页**不是**模拟点击翻页按钮,而是直接调用 Vue 实例上的 `pageChangeAction(page+1)`——干净、不触发用户交互动画、也不会被防自动化逻辑识别成"机器点击"。

**这个设计是整个扩展能在不打架的情况下"借力发力"的根本原因**——它不和 Boss 的 SPA 抢渲染权,而是把自己嵌入对方的状态机。

### 挂载点

```
document.body
└── #wrap (Boss Vue 根)
    └── .page-job-wrapper
        ├── #boss-helper-job  ◀── 筛选 UI (pages/zhipin/components/Ui.vue)
        ├── .job-search-wrapper
        └── .job-list

document.body
└── #boss-helper            ◀── 主面板 App.vue (浮动头像/配置入口)
```

两个 Vue 应用,各自 `createPinia()`、各自命名空间。这避免了 store 状态被 Boss 的页面切换重置。

### `elmGetter`:等元素出现/移除元素

`src/utils/elmGetter.ts` 用一个全局 `MutationObserver` 监听整个文档,新元素出现就检查是否匹配。所有"等岗位列表挂上来再 mount"、"移除广告"都基于它,而且**多个 selector 共用同一个 observer**(通过 `WeakMap<target, listener>`),省性能。

---

## 五、投递流水线 Pipeline

`src/composables/useApplying/index.ts` 的 `createHandle()` 用一个**嵌套数组的 DSL**描述复杂的"过滤 + 守卫"逻辑。

### DSL 形态

```ts
const pipeline = [
  h.communicated(),         // 简单 step
  h.jobTitle(),
  h.salaryRange(),
  [                          // ← 嵌套数组 = 「Group + Guard」
    async (args) => { ... 加载 card ... },   // 第一项是 guard
    h.activityFilter(),      // 后续 step 仅在 guard 通过时执行
    h.hrPosition(),
    h.jobAddress(),
    [                        // ← 再嵌套:高德地图组
      async (args, ctx) => { ... 调高德地图 ... },
      h.amap(),
    ],
    h.aiFiltering(),
    h.greeting(),            // ← 这里产生 after 钩子
  ],
]
```

### `compilePipeline` 编译规则

- 扁平 step → `before[]`,顺序执行,任一 throw 即中断
- 带 `after` 字段的 step → `after[]`,投递成功后再执行(如 AI 招呼语)
- 嵌套 `[guard, ...steps]` → guard 自动 `unshift` 到子序列前面,**只有当组内有 step 才会真正插入 guard**(配置全关时不浪费)

### 设计理念:贵的操作只在便宜的过滤都通过后再做

```
廉价过滤(纯字段判断,无 IO)
  communicated → jobTitle → company → salary → size
        │
        ▼ 全部通过
  ┌────────────────────────────────────────────────┐
  │ Guard: getCard()  ← 一次详情请求                │
  │   │                                             │
  │   ▼                                             │
  │  activity → hrPosition → jobAddress → ...       │
  │   │                                             │
  │   ▼ 全部通过                                     │
  │  ┌─────────────────────────────────────┐       │
  │  │ Guard: amapGeocode()  ← 高德 IO     │       │
  │  │  └─ amap()                           │       │
  │  └─────────────────────────────────────┘       │
  │   │                                             │
  │   ▼                                             │
  │  aiFiltering() ← 大模型调用,最贵的一步           │
  │  greeting()    ← 生成招呼语 (after 钩子)         │
  └────────────────────────────────────────────────┘
                       │
                       ▼ before 全跑完
                 sendPublishReq() ← 真正投递
                       │
                       ▼ 投递成功
                  执行 after[]
                  (发送 AI/自定义招呼语,通过 WebSocket)
```

### 配置开关 → 行为映射

`StepFactory` 的关键约定:**禁用即返回 `undefined`**。

```ts
const aiFiltering: StepFactory = () => {
  if (!conf.formData.aiFiltering.enable) return  // 关掉就不进 pipeline
  ...
}
```

`compilePipeline` 里 `if (h == null) continue` 会自动跳过。**配置项的开关不需要在 pipeline 数组里写 if-else**,而是在工厂函数里返回 undefined。配置 → 行为的映射极其干净。

新增一个筛选条件的步骤:

1. 在 `useApplying/handles.ts` 新增一个 `StepFactory`,内部读 `conf.formData.<key>`
2. 在 `useApplying/index.ts` 的 pipeline 数组里按位置插入(决定它在哪一组、哪一段)
3. 在 `stores/conf/info.ts` 新增默认值与 UI 元数据
4. 在 `pages/zhipin/components/Config.vue` 新增对应表单控件
5. 在 `composables/useStatistics.ts` 中新增统计字段(用于"今日因该过滤项被过滤的次数")

### 错误分类系统

`src/types/deliverError.ts` 定义了一组语义化错误,`useDeliver.ts` 据此采取不同动作:

| 错误类                      | 含义                      | 行为                              |
| --------------------------- | ------------------------- | --------------------------------- |
| `LimitError`                | 投递达到 Boss 硬上限(150) | 立即停止整个投递循环              |
| `RateLimitError`            | 触发 Boss 频率限制        | 自动把投递间隔 +3s,再 sleep 30s   |
| 其它 `BoosHelperError` 子类 | 各种过滤性失败            | 标记 warn/error 进日志,继续下一个 |
| 非 `BoosHelperError`        | 未预期错误                | 包装为 `UnknownError`,标记 error  |

**错误 = 流程信号**,这是把错误处理和流程控制统一抽象的范式。

### 投递循环主体

`src/pages/zhipin/hooks/useDeliver.ts` 的 `jobListHandle()` 是入口:

```
for each job in jobList._list:
    状态机: pending → wait → running → success/error/warn

    检查投递缓存(命中就标记 success/warn 跳过)

    for h of pipeline.before:
        await h({data}, ctx)   ← 任一抛错即中断

    sendPublishReq(data)      ← 真投递

    for h of pipeline.after:
        await h({data}, ctx)   ← 发招呼语(WS protobuf)

    写日志、更新统计、缓存结果
    delay(deliveryInterval)

达到 deliveryLimit → 自动停止
```

---

## 六、AI 双轨:OpenAI 兼容 vs 服务端代理

```
                    ┌──────────────────────────┐
                    │   配置 AI 模型             │
                    └────────────┬─────────────┘
                                 │
              ┌──────────────────┴────────────────┐
              ▼                                    ▼
   ┌────────────────────┐              ┌───────────────────────┐
   │  openai.Gpt        │              │  SignedKeyLLM         │
   │  (用户填 url+key)   │              │  (boss-helper.ocyss   │
   │                    │              │   .icu 服务端代理)     │
   │  ┌──────────────┐  │              │  ┌─────────────────┐  │
   │  │ background ? │  │              │  │ openapi-fetch + │  │
   │  └──────┬───────┘  │              │  │ Bearer signedKey│  │
   │     yes │  no      │              │  └─────────────────┘  │
   │     ▼   ▼          │              │                       │
   │  fetch via    fetch│              │  /v1/llm/invoke/      │
   │  background   in   │              │  greetings|filter     │
   │  (绕 CORS)    main │              │                       │
   └────────────────────┘              └───────────────────────┘
```

两条路径在 `aiFiltering()` 里通过 `model.getModel(curModel, prompt, vip)` 透明二选一:

- `vip=true` 走 `SignedKeyLLM`,业务方负责模型/简历/prompt 拼装,客户端只传岗位数据
- `vip=false` 走自建 OpenAI 兼容协议(用户填 url + api_key)。`other.background` 开关决定走 main-world 直接 fetch(简单)还是走 background fetch(绕 CORS)

`SignedKeyLLM.checkResume()` 在 step 工厂创建时就调用——是个**预热 / 资格校验**,如果用户简历没传上去,在投递开始前就抛错而不是投到一半才挂。

`useChat` 里的 `chatBossMessage()` 是 `onPrompt` 的回调,用来**把"我们要发的招呼语"塞进 chat 列表**作为时间轴展示——所以"AI 想了什么 → 发出去什么 → Boss 回了什么"在 UI 上是连续的对话流。

---

## 七、消息发送:寄生 Boss 的 WebSocket

最 hacky 的部分。Boss 自己的聊天用 protobuf over WebSocket。我们的招呼语怎么发出去?**绕开自己建连接,直接复用 Boss 自己的 client**:

```ts
// src/composables/useWebSocket/protobuf.ts
send() {
  if ('GeekChatCore' in window && window.GeekChatCore != null) {
    const client = window.GeekChatCore.getInstance().getClient().client
    client.send(this)                     // ← 直接拿 Boss 的 WS client
  } else if ('ChatWebsocket' in window) {
    window.ChatWebsocket.send(this)       // ← 旧版 fallback
  }
}
```

`Message` 类负责把内容编码成 `TechwolfChatProtocol` 格式的 protobuf:

- `mid = Date.now() + 68256432452609`(模拟 Boss 自己的消息 ID 算法)
- `type=1, body.type=1, templateId=1`(纯文本消息)

这意味着扩展**完全没有自己的 WebSocket 连接、不需要鉴权、不会被服务器识别为"非官方客户端"**——发出去的报文和 Boss 网页自己发的一模一样。

**代价**:要时刻跟着 Boss 升级 protobuf schema。`bak.ts` 的存在就是证据(以前用 MQTT 自己发,后来 Boss 改了,改成借用 `window.ChatWebsocket`)。注释里那段 `EventBus` 的失效记录("2025-12-22 失效,疑似 boss bug")也很说明问题——这条路径很脆弱,需要不断维护 fallback 链。

---

## 八、缓存:LRU + 处理器分级 TTL

`src/composables/usePipelineCache.ts` 的 `PipelineCacheManager` 不是简单的 KV 缓存,它按"信息源的稳定性"对 TTL 分级:

| processorType | TTL  | 含义                                |
| ------------- | ---- | ----------------------------------- |
| `aiFiltering` | 7 天 | AI 评分基本不会变,缓存最久          |
| `amap`        | 5 天 | 地址不变,距离也不变                 |
| `basic`       | 3 天 | 关键词匹配等基础结果,变动概率高一点 |

**只缓存非 error 状态**(`if (status === 'error') return`)——因为 error 通常是网络/未知错误,缓存了下次还会卡住;但 warn(主动过滤)缓存下来下次就能跳过省一次完整 pipeline。

LRU 兜底(`maxCacheSize: 10000`)+ 6 小时一次的过期清理。所有持久化都走 `counter.storageSet` 跨上下文落到 `chrome.storage.local`。

---

## 九、账号切换:cookies + 配置 + 统计三位一体

```
saveUser()                       changeUser(target)
   │                               │
   │  CookieInfo {                 │  1. saveUser(currentUid) ← 先把当前账号 form/统计存下来
   │    uid, user, avatar,         │  2. counter.storageSet(formDataKey, target.form)
   │    form: jsonClone(formData), │  3. setStatistics(target.statistics)
   │    statistics: ...            │  4. counter.cookieSwitch(target.uid)
   │  }                            │       └─ background:
   │                               │          ① 删除当前 zhipin.com 所有 cookie
   │  + 完整 cookies 数组          │          ② 写入 target.cookies
   │  → storage[local:conf-user]   │       (multi-tab 的 cookie 是全局的,
   │                               │        所以必须在 background 串行操作)
   ▼                               ▼
```

**设计理念**:**「账号 = cookies + 配置 + 统计」三位一体**。换号不仅换登录态,连"投递筛选条件"和"今日统计"一起换——避免把求 Java 的 prompt 用在求 Python 的账号上。

---

## 十、工程约定与陷阱

### Element Plus 用 `ehp` 命名空间

为了不和 Boss 自身 CSS 冲突,所有 `.el-xxx` 都被改成 `.ehp-xxx`。SCSS 通过 `additionalData` 在编译期注入 `$namespace: 'ehp'`,JS 用 `<ElConfigProvider namespace="ehp">`。**写自定义样式时只能用 `ehp-xxx`**。

### content_scripts 的 CSS 手工注入

`wxt.config.ts` 的 `build:manifestGenerated` hook 把 `/assets/main-world.css` 推进 manifest 的 content_scripts。原因:WXT 默认只给 `defineContentScript` 注入 CSS,但我们的 Vue 是在 main-world 里跑的——main-world 脚本通过 `<script>` 标签注入,不是 content script,WXT 不会自动注入 CSS。修改 main-world 样式入口时记得对齐这里。

### `oxlint + oxfmt` 而非 ESLint/Prettier

oxc 工具链,Rust 写的,比 ESLint 快一个数量级。`oxfmt` 配置 `sortImports.partitionByComment: true` 意味着 import 区块的注释会成为分组分隔符——给注释插队会改变排序,需要小心。`lint-staged` 在 commit 时自动跑 oxlint --fix 和 oxfmt。

### WXT 自动导入已关闭

`wxt.config.ts: imports: false`。`#imports` 仍可用于 WXT 自身导出(`defineBackground` / `defineContentScript` / `browser` / `storage`),但 Vue API、composable、组件都需要显式 import。

### manifest matches

扩展只在 `*://zhipin.com/*` 与 `*://*.zhipin.com/*` 注入。新增页面 hook 时:

1. 在 `main-world.ts` 的 `switch (router.path)` 里加 `case`,路径对齐 zhipin 路由
2. 在 `src/pages/<新页面>/index.ts` 实现 `run()` 入口

### 限额 120 vs 150

`useApplying/utils.ts` 的 `sendPublishReq` 里专门处理:

- **120 条**:Boss 弹"今日已与 120 位 BOSS 沟通"提示框 → **自动发确认请求 + cid:1 重试**(用户根本无感)
- **150 条**:硬上限 → 抛 `LimitError` 终止整个循环

这个细节是反复试出来的——一行代码背后是一次次踩坑。

### Pinia store 中的 TSX

`src/stores/log.tsx` 是 TSX 文件——项目启用了 `@vitejs/plugin-vue-jsx`,`tsconfig.json` 配置 `jsx: preserve` + `jsxImportSource: vue`。

### 跨上下文存储的 key 前缀

`counter.storageGet/Set/Rm` 的 key 默认前缀是 `sync:`。要用 `local:` / `session:` 必须显式带上前缀(见 `message/contentScript.ts` 的 `genKey()`)。

---

## 设计哲学一句话总结

1. **不重写,只寄生**——能用 Boss 自己的 Vue 实例、WebSocket client、page action,就绝不自己造。代价是脆弱(对 Boss 升级敏感),收益是行为不可区分于人类。
2. **DSL 化的 pipeline**——配置项的开关靠工厂返回 undefined 自然消失,贵的步骤靠嵌套数组的 guard 自然延后。配置 → 执行图的映射是结构性的,不是 if-else 堆出来的。
3. **三层 RPC 是无奈但优雅**——`comctx` + 自定义 Adapter 让 main-world 调 background 像调本地方法,把"在哪一层执行"这个工程细节藏掉了。
4. **错误是一等公民**——`BoosHelperError` 子类承载流程语义,`LimitError`/`RateLimitError` 不是失败而是"该停了/该慢点了"的信号。
5. **数据 + 配置 + 统计 = 账号**——换号是整体替换,避免跨账号的状态污染。
