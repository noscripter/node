# Node.js Core 全景架构与核心思想

> **分析基线**：`main` @ `7a11a9b2db0983fa65b52e91fd21814f1ed3c207`，源码版本 `v27.0.0-pre`，2026-07-15。当前仓库没有根级 `package.json`，也没有 `out/` 构建产物；因此本文描述的是该提交的**源码架构**，不是某个已安装或已运行二进制的动态观测结果。

> **在线版本**：[在 Lark 中阅读（含 5 张可缩放架构白板）](https://mayfairtech.larkenterprise.com/docx/TwXbd2wPnojRyoxHL7uc0um3nMe)

## 0. 新人上车：先把术语翻译成人话

这是一份导览，不是一场缩写词突击考试。你不需要先会 C++、V8 或操作系统内核；只要写过一点 JavaScript，就可以从这里出发。全文采用同一个心智模型：**把 Node.js 想成一家 24 小时餐厅**。类比负责让你先“看见”系统，紧跟其后的真实定义负责让你以后能读源码。笑话可以帮助记忆，但不能参与编译。

### 0.1 一张新人地图：Node.js 深夜餐厅

```mermaid
flowchart LR
  USER["食客：你的 JavaScript 程序"]
  API["前厅：fs、http、timers、streams 等 JS API"]
  WINDOW["员工窗口：internalBinding，把 JS 请求交给 C++"]

  subgraph ENV["一家完整分店：一个 Node Environment"]
    V8["V8 主厨：执行 JS、管理 heap、JIT 与 GC"]
    MANAGER["Node C++ 店长：管理进程、资源、生命周期和回调边界"]
    UV["libuv 领班：event loop、timer、I/O 与任务分派"]
    READY["出餐台：完成事件回到所属 loop，准备运行 JS callback"]
  end

  POLLER["OS 呼叫器：epoll / kqueue 报 readiness；IOCP 报 completion"]
  POOL["共享后厨：libuv worker pool 处理文件、系统 DNS 和部分 crypto/压缩"]
  WORKER["另一家分店：Worker 独立线程、Isolate、heap、loop 与 Environment"]
  V8BG["V8 维修班：部分可并发 GC/编译等引擎工作；不直接替你运行用户 callback"]

  USER --> API
  API -->|普通 JS| V8
  API -->|需要 native 能力| WINDOW --> MANAGER --> UV
  UV -->|网络等待| POLLER --> READY
  UV -->|可能阻塞的 native 工作| POOL --> READY
  READY -->|callback / Promise continuation| API
  USER -->|CPU 密集 JS| WORKER
  WORKER -->|MessagePort / structured clone| API
  V8 --> V8BG
```

可独立复用的图源位于 [`beginner-mental-model.mmd`](beginner-mental-model.mmd)。

这张图只表达责任关系，不表达精确时间顺序。最重要的第一印象是：**event loop 不是厨师，线程池也不执行你的 JavaScript callback**。V8 执行 JS；libuv 和 OS 帮它等待与完成 I/O；Node 把完成结果安全地送回正确的 JS 世界。

### 0.2 V8 与 JavaScript VM：厨房里到底有什么

| 术语 | 先用人话理解 | 准确定义与边界 |
| --- | --- | --- |
| **V8** | Node 的“JS 主厨”。它懂 JavaScript，不会自己开文件、监听端口或实现 `node:fs`。 | Google 的 JavaScript/WebAssembly 引擎，负责解析与执行代码、对象内存、JIT、GC、Isolate、Context 和 microtask。Node 是 V8 的宿主，不是 V8 的别名。 |
| **VM / JavaScript engine** | 一台能读懂并执行 JS 的机器；不是一台真的虚拟电脑，也不会在桌下多长一块 CPU。 | 提供语言语义、对象模型、执行栈和内存管理的运行环境。V8 是 Node 使用的 JS engine。 |
| **Call stack** | 主厨手里的“当前步骤单”：函数调用一层压一层，返回时一层层划掉。 | 保存正在执行的函数帧、局部状态和返回位置。同步死循环会一直占着它，所以 event loop 再勤快也插不上话。“primitive 一定在 stack、object 一定在 heap”只是过度简化；优化后的 V8 可把值放在寄存器、stack slot 或 heap context。 |
| **Heap** | 厨房仓库：对象、数组、闭包等需要跨调用存活的数据通常在这里。 | Isolate 管理的动态 JS 对象图，但它不是 Node process 的全部内存；Buffer backing store、C++ 对象、OpenSSL、线程 stack 等可能在 V8 heap 之外。它也不同于 OS socket/file handle。 |
| **JIT** | 主厨发现“蛋炒饭”被点了十万次，就把常走步骤改成高速专线；原先假设失效时还会撤回优化，多少有点现实主义。 | Just-In-Time compilation。V8 可先解释 bytecode，再依据运行反馈选择更高优化层并生成机器码；假设失效时会 deoptimize。不是每个函数都走固定层级，也不是把整个 Node 程序一次性编译完。 |
| **GC、root 与 reachability** | 自动收盘员从仍有效的订单和取货牌出发盘点仓库；还能沿引用链找到的东西，它不敢扔。GC 会追引用，不会读产品需求。 | Garbage Collection 从 stack、global、强 V8 handle 等 roots 追踪对象，回收不可达 heap 内存；循环引用只要整体不可达也可回收。回收后内存不保证立即归还 OS，GC 也不会替你正确关闭 socket/native handle。 |
| **V8 Handle / HandleScope** | JS 对象会被 GC 搬家，C++ 不能把旧门牌号刻在手上；V8 handle 是会随仓库更新的取货牌，HandleScope 批量管理短期取货牌。 | GC-aware 的 JS value reference；`Local` 通常只在 scope 内有效，`Global` 可跨 scope。它与 libuv 的 socket/timer handle、OS handle/file descriptor 是三种同名不同物。 |
| **Isolate** | 一间上锁的独立厨房，拥有自己的仓库。隔壁 Worker 不能直接伸手来拿你的普通 JS 对象。 | 一个 V8 引擎实例及其普通 JS object graph、GC 和引擎状态。主线程和每个 Worker 通常各有一个；它不是 OS process 或安全沙箱。普通对象不跨 Isolate 直接引用，但 `SharedArrayBuffer` backing store 是需用 Atomics 协调的显式共享例外。 |
| **Context** | 同一厨房里的独立用餐区：有自己的 `globalThis` 和一套 `Object`、`Array` 等内建对象。 | V8 对“global object + 一组 built-ins/intrinsics”的封装。一个 Isolate 可以有多个 Context；`vm.createContext()` 创建 Context，但它不是完整 Node 实例，也不是运行不可信代码的防弹沙箱。 |
| **Realm** | Context 是用餐区；ECMAScript realm 是它的全局对象和标准餐具；`node::Realm` 是 Node 给合资格用餐区配的营业档案袋。 | ECMAScript Realm 是语言规范概念，`v8::Context` 是 V8 的主要表示，Node `Realm` 再增加 per-realm builtins/bindings、cleanup hooks 和 `BaseObject`。三者相关但不能机械画等号；普通 `vm.Context` 当前不自动拥有 Node Realm。 |
| **Intrinsic / primordial** | JS 世界出厂自带的餐具，如 `Array`、`Promise` 及其原型方法；`primordials` 是内核提前锁进员工柜的干净餐具。 | Intrinsic 是语言环境的内建对象。用户能 monkey-patch 全局原型，Node bootstrap 因而保存安全引用供 `lib/internal` 使用，避免用户改写内核所依赖的语义。 |
| **Promise** | 一张未来取餐号，不是厨师。把 CPU 死循环包进 `new Promise()`，只会得到一张看着很异步的堵车罚单。 | 表示未来结果及 fulfilled/rejected 传播；constructor executor 同步执行，`.then/.catch/.finally` reactions 和 `await` continuation 通常通过 microtask 推进。Promise 本身不创建线程。 |
| **Microtask** | 贴在服务员袖口的“小纸条”：当前 JS 边界结束到达 checkpoint 时，先处理这些，再接下一批普通事件。Promise continuation 常在这里。 | 当前 Context 关联的 V8 MicrotaskQueue 中的工作，例如 Promise reactions 和 `queueMicrotask()`。它不打断当前函数；“micro”的是排队规则，不是工作量，递归追加同样可能让后续 I/O 饿肚子。多个可同步互访的 Context 还可以共享队列。 |
| **`process.nextTick()`** | Node 自己另设的“更急纸条”。名字像“下一轮”，实际上通常在当前操作结束后、继续事件循环前优先清空。 | Node 管理的 per-Environment queue，不是 V8 microtask queue。不要把 Node 调度粗暴画成只有“宏任务/微任务”两条队列；具体 checkpoint 由 Node 与 V8 共同建立。 |
| **Foreground / background task** | 前台任务必须回到这间厨房的正确工作台；后台维修班可做部分 GC/编译辅助工作，却不能随便端走你的用户 callback。 | V8 Platform 可在后台线程处理部分可并发的引擎工作，并把必须进入目标 Isolate 的 foreground task 投回其所属线程。这里的 foreground 对 Worker 而言就是 Worker 线程，不必然是 process 主线程。 |
| **Startup snapshot** | 构建时把厨房基础备料先封成标准备餐包；开店时恢复，而不是每次从洗菜开始。今天顾客的 argv/env 不能冻进去。 | 序列化的可恢复 V8 heap object graph 与 Node metadata，运行时 deserialize 后修补本次运行关联。它不是运行中进程休眠，也不同于诊断内存的 heap snapshot。 |

把 V8 的执行过程粗略画成一条线，是：**JS source → parser/AST → Ignition bytecode → 解释执行 → 根据反馈选择可用的优化层 → machine code**。当前 V8 还可能使用 Sparkplug、Maglev、TurboFan 等不同层级，但这不是每个函数必经的固定旅游路线；平台、开关、热点程度和类型反馈都会改变路径，优化假设失效时还会 deopt。记住因果关系比背引擎品牌名更重要：**先尽快跑起来，再把值得优化且假设稳定的路径跑快。**

### 0.3 Node、libuv 与异步：谁在等，谁在干活

| 术语 | 先用人话理解 | 准确定义与边界 |
| --- | --- | --- |
| **Runtime / host** | V8 只会做菜；Node 还提供餐厅、门牌、采购、消防和顾客规则。 | 宿主把语言引擎、OS 能力、标准 API、模块系统和生命周期组合成可运行程序的环境。Node.js 是 runtime，V8 是其中的 engine。 |
| **Event loop** | 领班反复巡视：“计时器到点了吗？网络有消息吗？后厨做完了吗？现在该叫哪个 callback？”没事时它会等待 OS 通知，不是喝着浓缩咖啡疯狂空转。 | 每个 `uv_loop_t` 管理 timer、poll、handle 和 request completion。Node 还在 `uv_run()` 周围协调 V8 tasks、nextTick、microtasks、`beforeExit` 与退出；一个普通 JS callback 未返回前，另一个不会随意抢占它。 |
| **Callback** | 交给别人的一张“之后请调用这个函数”任务卡；至于“之后”是三秒后还是下一行，得看接卡的人。 | Callback 只是作为参数传递的函数，可能同步执行（如 `Array#map`、`EventEmitter#emit`），也可能在事件完成后异步执行。libuv pool 完成 native 工作后，用户 callback 仍回所属 JS 线程。 |
| **Timer** | `setTimeout(fn, 100)` 是“100ms 后最早可以叫号”，不是“第 100ms 必须破门而入”。店长若正被死循环扣住，号码牌只能礼貌迟到。 | Timer 设置最早可运行时间，不保证精确 deadline，也不创建线程。Node 用少量 native timer 驱动大量 JS timer，而不是每个 timer 配一条线程。 |
| **Non-blocking** | 点单后服务员可以继续服务别人；不代表这道菜不耗时间，也不代表厨房里没有线程在干活。 | 调用方不必同步等待完成。底层可能使用平台 I/O 通知、线程池或别的异步机制；CPU 密集 JS 仍会阻塞所属 JS 线程。 |
| **Concurrency / parallelism** | 并发是一个服务员同时照看多桌；并行是确实多开了一间厨房。两者经常一起出现，但不是同一个词。 | Concurrency 表示多个工作在时间上交错推进；parallelism 表示多个执行资源同时工作。Node 主线程 I/O 多为并发，Worker 可提供并行 JS。 |
| **OS I/O notification** | 网络呼叫器通知“现在可以处理了”，但 Unix 与 Windows 的口音不同：epoll/kqueue 主要说“可读/可写”，IOCP 更常说“这次操作完成了”。 | Unix 常用 readiness，Windows IOCP 更偏 completion；libuv 抹平平台差异并把事件交回所属 loop。通知机制不执行 JS，也不是 libuv worker pool。 |
| **libuv** | 跨平台领班：对 Linux、macOS、Windows 各说各的方言，对 Node 统一汇报。 | Node 使用的 C 库，提供 event loop、timer、TCP/UDP/pipe/TTY、process、signal、FS/DNS 抽象、线程与全局 worker pool。 |
| **Handle** | 长期营业设备，例如一直监听的电话或定时叫号器；通常要显式关闭。 | libuv 中相对长寿、可 active/ref/close 的对象，如 TCP server、timer、pipe。referenced active handle 通常能让 loop 保持存活；它不等于 V8 handle，也不保证与 OS file descriptor 一一对应。 |
| **Request** | 一张一次性工单，例如“读这个文件”；完成后工单结束。 | libuv 中一次异步操作的短寿命对象，如 `uv_fs_t`。handle 和 request 生命周期不同，所以 Node 分别有 `HandleWrap` 与 `ReqWrap`。 |
| **`ref()` / `unref()`** | `ref` 是“这桌没结账，不能打烊”；`unref` 是“有空就做，但别只为我整夜开店”。 | 控制某些 handle 是否单独保持 event loop 存活；它不等于取消任务、释放对象或提高优先级。 |
| **libuv worker pool** | 共享后厨，默认四位帮工；文件、系统 DNS、部分 crypto/压缩可能在这里排队。帮工不会跑你的 JS callback。 | 进程级共享线程池，服务 main loop、Workers 和 addon 提交的相关 native work。调大 `UV_THREADPOOL_SIZE` 只影响使用该 pool 的路径。 |
| **Worker / `worker_threads`** | 真正再开一家分店：另有线程、厨房、仓库和领班。开店有成本，沟通要传单，不能隔墙抢对象。 | 独立 OS thread、Isolate、heap、event loop、Environment 和模块缓存，可并行执行 JS；通过 MessagePort、structured clone、transfer 或显式共享内存通信。它不是 libuv pool，也不是 Node 自动替你管理的通用任务池。 |
| **Environment** | 一家可独立营业和打烊的 Node 分店档案：厨房、领班、主营业区及 timer/inspector/退出状态成套出现。 | Node.js 实例边界，关联一个 event loop、一个 Isolate 和一个 principal Realm，并持有大量 per-instance 状态。它不是 `process.env`，也不等于整个 OS process。 |
| **`NodeMainInstance`** | 普通 CLI 主分店的开业经理：把主厨房、门店档案和领班组起来，营业结束后负责收尾。 | 主进程路径中创建主 Isolate/IsolateData/Environment、加载 JS 并进入主循环的 C++ 编排对象。Worker 也创建 Environment，但不经这位主店经理。 |
| **Binding / `internalBinding()`** | JS 前厅与 C++ 后场之间的员工窗口；`internalBinding()` 上贴着“顾客止步”。 | Binding 把 native C++ 函数/对象暴露给 JS。`internalBinding()` 是 core-private loader，不是公共用户 API，也不承诺兼容。 |
| **`BaseObject` / `AsyncWrap`** | JS 取餐牌和 native 锅具要成对登记；还要记住这张异步工单是谁触发的，否则上下文会像外卖地址一样丢掉。 | `BaseObject` 连接 V8 object 与 C++ 实例；`AsyncWrap` 增加 async/trigger id、callback scope 和诊断上下文，子类协调 handle/request 的引用与清理。 |

### 0.4 模块、构建与兼容：菜谱怎样进入餐厅

| 术语 | 先用人话理解 | 准确定义与边界 |
| --- | --- | --- |
| **Bootstrap** | 开门前的准备：通电、摆餐具、登记员工窗口。它不是 Bootstrap CSS，前端同学先把手从 CDN 链接上拿开。 | Node 在用户代码前建立 primordials、binding loader、builtins、`process`、globals、task hooks 和运行模式的启动过程。 |
| **Builtin / core module** | 随餐厅一起交付的官方菜谱，如 `node:fs`；不是启动时临时去 `node_modules` 买来的。 | Node 自带模块。大部分核心 JS 由 `js2c` 嵌入 binary，由 `BuiltinModule` 在 bootstrap 后加载；公共 builtin 与 `lib/internal` 的稳定性不同。 |
| **CJS / CommonJS** | `require()` 像同步取一本菜谱：拿到 exports 才继续，历史包袱也一并打包。 | Node 传统模块系统，支持同步加载、`module.exports`、`require.cache`、JSON 和 `.node` addon 等语义。 |
| **ESM** | 先登记整桌宴席的依赖关系，再链接和求值；可有 top-level await。 | 标准 ECMAScript module graph，经历 resolve、load/translate、link/evaluate，使用 URL 语义并支持同步/异步 customization hooks。 |
| **Loader** | 图书管理员兼路线规划器：先回答“你说的是哪本书”，再决定怎么读、缓存和执行。 | 把模块标识符解析为资源并完成装载/编译/求值的机制。Builtin、CJS、ESM loader 分开，是因为启动信任边界和语言语义不同。 |
| **Resolve / load / link / evaluate** | 找地址、取内容、把依赖接线、最后通电运行；不要把四步都叫“import 了一下”。 | ESM 生命周期的不同阶段。区分它们有助于定位“找不到模块”“格式不识别”“循环依赖”“运行异常”分别发生在哪。 |
| **`js2c`** | 把 JS 菜谱封进建筑材料，开店时不必先靠 `fs` 去读实现 `fs` 的文件；这能避开一只很尴尬的启动鸡。 | 构建工具 `node_js2c` 把 `lib/**/*.js`、选定依赖 JS 和配置生成 `node_javascript.cc`，供 `BuiltinModule` 从 binary 读取。 |
| **GYP / target graph** | 建筑总图：哪些源文件先加工、哪些工具先造、最后怎样拼成 `node`。Make/Ninja/MSBuild 是不同施工队，不是不同建筑。 | `node.gyp` 等描述官方构建目标及依赖，GYP 生成平台 backend。`BUILD.gn` 在本仓库是辅助、非官方发布路径。 |
| **Native addon** | 给餐厅接一台自制机器的 `.node` 动态库。接标准插座最省心；直接焊主板性能自由度高，升级时也更刺激。 | 用 C/C++ 扩展 Node 的共享库，经 `process.dlopen()` 加载。可使用稳定 Node-API，或直接依赖 V8/Node C++ API。 |
| **API** | 菜单写着“可以点什么、怎么点”。 | 源代码层面对调用者暴露的名称、参数和行为契约。公共 JS API 的兼容由文档稳定等级、弃用政策和 SemVer 约束。 |
| **ABI** | 插头形状、电压和针脚布局：菜单名字没变，插头不合也照样开不了机。 | 已编译二进制之间的调用约定、符号和对象布局契约。直接 V8/Node C++ addon 容易随版本产生 ABI 变化。 |
| **Node-API** | 官方标准插座；尽量让 addon 跨 Node major 不重焊。 | 引擎无关的稳定 C ABI，以不透明句柄隔离 V8 细节。它不是 addon 本身，也不同于 C++ 包装库 `node-addon-api`；若同时调用 V8/libuv/Node C++ 私有接口，稳定保证会被削弱。 |
| **`NODE_MODULE_VERSION`** | 直接焊主板方案的“插头代号”；代号不匹配，门卫 `process.dlopen()` 会拒绝放行。 | Node 原生模块 ABI 版本。当前源码值为 147；它不等于 Node-API version。 |
| **SemVer / stability** | 版本号是合同摘要，不是许愿池；Experimental 菜单仍可能换做法。 | 语义化版本与 Node 文档稳定等级共同描述公开行为的变更约束。internal API、experimental API、Embedder API 的承诺不同。 |
| **Embedder** | 把整家 Node 餐厅开进另一栋 C++ 商场，而不是只写一个 npm 包。 | 使用 `libnode`/C++ API 在宿主应用中创建 Platform、Isolate、Environment 并运行 Node。该 API 能力强，但 major 版本可发生 breaking change。 |
| **Component / end-to-end / soak / production evidence** | 试吃一勺汤、完整吃一桌、连续营业七天、真实门店营业额，是四种证据，不能互相冒充。 | 组件测试/基准只覆盖执行到的局部路径；端到端验证覆盖完整链路；soak 关注长时稳态；production 才是实际环境证据。 |

四个很像“快照”的产物尤其容易串台：

| 名称 | 保存什么 | 用来做什么 | 不是什么 |
| --- | --- | --- | --- |
| `node_javascript.cc` 内嵌源码 | core JS 源码和配置生成的数据表 | 让 `BuiltinModule` 不依赖启动时磁盘读取 | 不是 JIT 机器码，也不是正在运行的 heap |
| Builtin code cache | V8 对 builtin 编译结果的配套缓存 | 减少重复解析/编译工作 | 不是根据生产热点永久保存的最高层 JIT 代码 |
| Startup snapshot | 可序列化的预初始化 V8 object graph 与 Node metadata | 加快进程启动并恢复 bootstrap 基础状态 | 不是把 socket、顾客请求和构建机环境一起速冻 |
| Diagnostic heap snapshot | 某时刻的 JS heap 对象图报告 | 分析谁占内存、谁引用谁 | 不是启动材料；它是仓库盘点报告，不是中央厨房备餐包 |

图里还会出现一些缩写，先把身份证发给它们：

| 缩写 | 展开与用途 |
| --- | --- |
| REPL | Read–Eval–Print Loop：读一段代码、执行、打印结果、继续等待，俗称交互式 Node 控制台。 |
| SEA | Single Executable Applications：把应用资源注入单个可执行文件的交付方式，不等于 startup snapshot。 |
| TLA | Top-Level Await：ESM 顶层可使用的 `await`；它会影响模块图求值，不是 CJS 的同步 `require()`。 |
| IPC | Inter-Process Communication：进程间通信；Worker 的 MessagePort 是线程间消息机制，不要仅因“都在发消息”就混成一个实现。 |
| TLS | Transport Layer Security：HTTPS 等安全传输的加密协议层，Node 主要借助 OpenSSL/ncrypto 实现。 |
| DNS | Domain Name System：域名解析；`dns.lookup()` 与 `dns.resolve*()` 在 Node 内部走的调度路径并不相同。 |
| TTY | 终端设备抽象；交互式 stdin/stdout、颜色和窗口尺寸等行为会涉及它。 |
| Wasm | WebAssembly：V8 可执行的紧凑二进制指令格式，不是 Node 的第三种模块 loader。 |
| WPT | Web Platform Tests：验证 Web 标准兼容行为的共享测试集合；通过它不自动证明 Node 的全部系统行为。 |

### 0.5 用十行代码看懂“排队”

下面示例假设它是普通 CommonJS 文件。先运行同步代码；当前调用边界结束后，Node 的 `nextTick` queue 通常先于 V8 Promise microtask；timer 要等事件循环进入相应阶段。

```js
console.log('1 同步：先点单');
setTimeout(() => console.log('4 timer：叫号器到点'), 0);
Promise.resolve().then(() => console.log('3 microtask：袖口小纸条'));
process.nextTick(() => console.log('2 nextTick：更急的小纸条'));
```

通常输出 `1 → 2 → 3 → 4`。这里的重点不是背诵一张永恒不变的“队列优先级表”，而是学会问三个问题：**谁把工作排进哪条队列？checkpoint 在哪里？最后由哪个线程进入 JS？** ESM 顶层求值本身处于不同调度上下文，顺序细节可能不同；所以源码分析应沿真实入口走，而不是拿一条口诀走遍天下。

再看一个常被名字骗到的例子：

```js
setTimeout(() => console.log('如果餐厅还开着，我仍然会执行'), 1_000).unref();
```

`unref()` 没有取消 timer，只是告诉 event loop：“别只为等我而继续保持进程存活。”如果还有别的 referenced work，timer 仍可能正常触发。它是关店投票，不是订单碎纸机。

### 0.6 类比什么时候必须放下

餐厅类比适合建立责任边界，不适合推导精确调度顺序、锁、内存布局或 ABI。进入后续章节后，看到粗体英文术语时以右侧真实定义和源码锚点为准。最有用的阅读习惯不是“这个名词像什么”，而是连续追问：**谁拥有它、谁创建它、谁能并发访问它、谁让它保持存活、最后谁清理它？**

## 1. 一句话结论

如果只记住一句话：**Node.js 是餐厅总经理，不是某一位厨师。**它把 V8、libuv、C++ 核心和内嵌 JavaScript 编排成一个完整 runtime：V8 执行 JavaScript/WebAssembly 并管理 JIT、GC 与 microtask；libuv 负责跨平台事件循环、非阻塞 I/O 和共享 worker pool；Node C++ 核心管理进程、VM、线程、原生资源与生命周期；内嵌 JavaScript 则实现公共 API、模块系统、兼容策略和调度语义。

从第一性原理看，V8 只懂语言执行，不自带 `fs`、socket、进程或 Node 模块语义；libuv 会和操作系统打交道，却不认识 JavaScript 对象。单独请主厨或单独请领班都开不了餐厅。Node.js 的核心价值，是把它们连接成一个有稳定 API、可诊断、可扩展、可跨平台发布的宿主环境。

本文把项目分成五个相互咬合的部分：

1. **构建期组装**：Python/GYP 把 C++、内建 JS、依赖库和启动快照装配成 `node`。
2. **实例与生命周期**：`Environment` 把一个事件循环、一个 V8 Isolate 和一个 principal Realm 绑定成 Node 实例。
3. **JS 策略层与 native 机制层**：公共 JS API 和 `lib/internal` 通过 `internalBinding()` 调用 C++ binding。
4. **异步与并发**：平台 I/O 通知、libuv 全局线程池、Worker 独立 Isolate、V8 后台线程各有不同职责。
5. **兼容与保障**：公共 JS、Node-API、直接 C++ addon、Embedder API 和 core-private API 有不同稳定性边界，测试、基准和文档分别证明不同性质的结论。

## 2. 运行时总架构图

下图的阅读方向是“用户语义 → Node 内核 → 引擎/依赖 → OS”。它也把主 JS 线程、libuv 线程池、Worker 和 V8 后台线程明确分开。

```mermaid
flowchart TB
  INVOKE["进程入口：node app.js / -e / REPL / test / watch / SEA / C++ Embedder"]
  USER["用户代码：CommonJS、ESM、Web API、业务回调"]
  ADDON["原生扩展 .node：补充本地能力或复用原生库"]

  subgraph LIFECYCLE["C++ 生命周期与实例编排（src/）"]
    START["node::Start / StartInternal：解析选项，初始化 OpenSSL、NodePlatform 与 V8"]
    MAIN["NodeMainInstance：创建主 Isolate、IsolateData、Environment"]
    INSTANCE["Node 实例边界：1 Environment + 1 uv_loop + 1 Isolate + 1 principal Realm"]
    LOOP["SpinEventLoopInternal：uv_run、DrainTasks、beforeExit、exit 与清理"]
    WORKER["Worker：独立 OS 线程、uv_loop、Isolate、Environment 与模块缓存"]
    MSG["MessagePort / structured clone / SharedArrayBuffer：显式跨 Worker 通信"]
    DIAG["Inspector、async_hooks、diagnostics_channel、trace_events、perf_hooks、report"]
  end

  subgraph JSLAYER["JavaScript 语义与策略层（lib/）"]
    BOOT["Bootstrap：primordials、realm.js、node.js、pre_execution"]
    BUILTIN["BuiltinModule：装载已嵌入二进制的核心 JS，不依赖启动时磁盘 I/O"]
    LOADERS["用户模块装载：CJS 与 ESM 的 resolve、cache、hooks、compile/evaluate"]
    PUBLIC["公共核心 API：fs、net、http、streams、timers、crypto、worker、fetch 等"]
    INTERNAL["lib/internal：参数校验、策略组合、兼容逻辑与内部状态"]
    TASKS["调度交汇：timer/immediate、Node nextTick queue、V8 microtask queue 与 rejection"]
  end

  subgraph BRIDGE["JS ↔ Native 边界与对象生命周期"]
    IB["internalBinding(name)：私有核心 JS 到 C++ binding registry 的桥"]
    WRAP["BaseObject → AsyncWrap → HandleWrap / ReqWrap：绑定 JS 对象、原生资源和异步因果"]
    NAPI["process.dlopen + Node-API：推荐的稳定原生扩展 ABI"]
    DIRECT["直接 V8 / Node C++ Addon ABI：能力更底层，但随 NODE_MODULE_VERSION 变化"]
  end

  subgraph NATIVE["执行引擎、跨平台 I/O 与专用原生库（deps/）"]
    V8["V8：ECMAScript / WebAssembly、JIT、GC、Isolate、Context 与 microtask queue"]
    UV["libuv：事件循环、socket poll、timer、process、FS/DNS 抽象与全局线程池"]
    PROTOCOL["OpenSSL / ncrypto、llhttp、nghttp2、c-ares：TLS、加密与网络协议"]
    DATA["ICU、Ada、simdutf、zlib/Brotli/zstd、SQLite：文本、URL、压缩与数据能力"]
    JSDEPS["Undici、Amaro 等嵌入 JS 依赖：fetch/WebSocket 与 TypeScript type stripping"]
  end

  subgraph RESOURCES["实际并发资源与操作系统"]
    POLLER["OS I/O 通知：epoll/kqueue readiness；IOCP completion；JS 回调仍在所属 loop 线程"]
    UVPOOL["libuv 全局 worker pool：FS、系统 DNS、部分 crypto/压缩和 uv_queue_work"]
    V8BG["V8 Platform 后台线程：部分可并发 GC/编译等引擎任务，不直接运行用户回调"]
    OS["OS：文件、socket、进程、信号、终端与系统时钟"]
  end

  INVOKE --> START --> MAIN --> INSTANCE
  INSTANCE --> BOOT --> BUILTIN --> LOADERS --> USER
  USER --> PUBLIC --> INTERNAL
  INTERNAL --> TASKS
  INTERNAL --> IB
  PUBLIC --> IB
  ADDON --> NAPI
  ADDON --> DIRECT
  NAPI --> INSTANCE
  DIRECT --> V8
  IB --> WRAP
  WRAP --> INSTANCE
  WRAP --> UV
  IB --> V8
  IB --> UV
  IB --> PROTOCOL
  INTERNAL --> JSDEPS
  TASKS --> LOOP
  INSTANCE --> LOOP
  INSTANCE --> V8
  INSTANCE --> UV
  INSTANCE --> DIAG
  WRAP --> DIAG
  USER -->|new Worker| WORKER
  INSTANCE -->|发送到主线程| MSG
  MSG -->|送达主 Environment| INSTANCE
  MSG -->|发送到子线程| WORKER
  WORKER -->|送达 Worker| MSG
  WORKER -->|每个 Worker 重复实例边界| INSTANCE
  UV --> POLLER
  UV --> UVPOOL
  V8 --> V8BG
  PROTOCOL --> OS
  DATA --> OS
  POLLER --> OS
  UVPOOL --> OS
  LOOP -->|完成回调进入 JS| TASKS
```

可独立复用的图源位于 [`runtime-architecture.mmd`](runtime-architecture.mmd)。

## 3. 最重要的所有权模型

理解 Node.js 源码最有效的方法，不是先记几百个文件，而是先区分以下对象各自拥有的状态。

| 边界 | 它是什么 | 主要用途与所有权 |
| --- | --- | --- |
| Process | 一个宿主进程 | 持有一次性初始化的 V8 Platform、全局选项、信号/stdio、OpenSSL 状态和内建 binding 注册表。 |
| `MultiIsolatePlatform` | Node 对 `v8::Platform` 的实现 | 为多个 Isolate 调度 V8 foreground task 与可并发的引擎后台工作；Worker 能共享进程级 Platform。 |
| OS thread | 实际执行线程 | 主线程和每个 Worker 都有线程亲和的事件循环；同一个 V8 Isolate 不能被任意线程无锁并发进入。 |
| `v8::Isolate` | 一台独立 JS VM | 拥有普通 JS object graph、GC 和引擎状态。主线程与每个 Worker 各自有一个，彼此的普通对象不能直接引用；显式共享的 `SharedArrayBuffer` backing store 是例外。Isolate 是隔离边界，不是安全沙箱或 OS process。 |
| `IsolateData` | Node 的 per-Isolate 数据 | 保存快速字符串表、builtin loader、Platform/loop 关联和若干模板；Embedder 可在同一 Isolate 的合法范围内复用它。 |
| `v8::Context` | 一个 global 与一套 intrinsic | `vm.Context` 可在同一 Isolate 中创建多个 Context；Context 不等于完整 Node 实例，`node:vm` 也不是运行不可信代码的安全机制。 |
| `Realm` | Node 对 ECMAScript realm 的包装 | 保存 per-realm builtin/binding 状态、cleanup hooks 和 `BaseObject` 列表。principal Realm 对应 Environment 的主 Context；当前普通 `vm.Context` 不创建 Node Realm。 |
| `Environment` | 一个 Node.js 实例 | 关联**一个** `uv_loop_t`、**一个** Isolate 和**一个** principal Realm，并保存 timer、async hooks、inspector、process 与退出状态。 |
| `BaseObject` | JS 对象与 C++ 对象的桥 | 把 V8 object 的 internal field 指向 C++ 实例，并协调 JS GC 与 native 生命周期。 |
| `AsyncWrap` / `HandleWrap` / `ReqWrap` | 异步资源包装层 | 增加 async id/trigger id、native→JS callback scope、libuv handle/request 的 ref、close 和清理语义。 |

这套分层解决的是一个基本矛盾：V8 的对象生命由 GC 决定，而 socket、文件请求、timer 等原生资源的生命由 OS 与 libuv 决定。如果只依赖 GC，活跃 socket 可能被过早回收；如果只依赖 native 引用，JS 对象又会泄漏。`BaseObject` 及其子类把两种生命周期显式对齐。

源码锚点：[`src/README.md`](../src/README.md#L278-L408) 定义 Context、event loop、Environment、Realm、IsolateData 和 Platform；[`src/README.md`](../src/README.md#L917-L1078) 定义 BaseObject/AsyncWrap/HandleWrap/ReqWrap。

## 4. 从进程启动到退出

```mermaid
flowchart TD
  ENTRY["OS 入口：main / wmain"]
  START["node::Start → StartInternal：参数、进程级初始化与运行模式分流"]
  ONCE["InitializeOncePerProcess：注册 bindings，初始化 OpenSSL、NodePlatform 与 V8"]
  INSTANCE["NodeMainInstance：默认 uv loop + platform + args"]
  ENV["创建主 Isolate、Context、IsolateData、Environment 与 principal Realm"]

  SNAP{"是否使用 embedded snapshot？"}
  RESTORE["反序列化预初始化 V8 heap、bootstrap state 与 code cache"]
  COLD["冷启动：primordials → realm.js → node.js → web/thread/process switches"]
  READY["BuiltinModule 与 internalBinding 可用"]
  LOADENV["LoadEnvironment + pre_execution：注入本次运行的 argv、env、权限、IPC、preload 与 hooks"]

  MODE{"StartExecution 选择入口"}
  CJS["CommonJS：resolve → cache → wrap → 同步执行"]
  ESM["ESM：resolve → load/translate → link/evaluate module graph"]
  OTHER["其他模式：eval、REPL、Worker、test runner、watch、SEA 或 embedder callback"]
  USER["用户代码注册 handle、request、timer、microtask 或 Worker"]

  SPIN["SpinEventLoopInternal"]
  UV["uv_run：等待并分派 timer、I/O 与 native completion"]
  CALLBACK["native → JS callback scope：在所属 event-loop 线程运行用户 callback"]
  CHECK["callback 返回边界：处理 Node nextTick、V8 microtasks 与 rejection"]
  V8TASKS["platform → DrainTasks：排空该 Isolate 的 V8 foreground tasks"]
  ACTIVE{"仍有 referenced work？"}
  BEFORE["触发 beforeExit"]
  REENTER{"beforeExit 是否又创建 referenced work？"}
  EXIT["触发 exit，确定 exit code"]
  CLEAN["按反向依赖清理：Workers → handles → hooks → Realm → Environment → Isolate → Platform"]
  DONE["进程退出"]

  ENTRY --> START --> ONCE --> INSTANCE --> ENV --> SNAP
  SNAP -->|是| RESTORE --> READY
  SNAP -->|否| COLD --> READY
  READY --> LOADENV --> MODE
  MODE -->|CJS| CJS --> USER
  MODE -->|ESM| ESM --> USER
  MODE -->|其他| OTHER --> USER
  USER --> SPIN --> UV
  UV -->|分派 callback| CALLBACK --> CHECK --> V8TASKS
  UV -->|本轮返回| V8TASKS --> ACTIVE
  ACTIVE -->|是| UV
  ACTIVE -->|否| BEFORE --> REENTER
  REENTER -->|是| UV
  REENTER -->|否| EXIT --> CLEAN --> DONE
```

可独立复用的图源位于 [`startup-sequence.mmd`](startup-sequence.mmd)。

### 4.1 Native 入口很薄，真正入口在 `StartInternal`

Unix `main()` 最终只调用 `node::Start(argc, argv)`；Windows 先把宽字符参数转为 UTF-8，再汇合到同一入口。`StartInternal()` 完成 argv 设置、一次性进程初始化、SEA/snapshot 分支，并用 `uv_default_loop()`、NodePlatform 和参数构造 `NodeMainInstance`。这条链可从 [`src/node_main.cc`](../src/node_main.cc#L34-L98)、[`src/node.cc`](../src/node.cc#L1564-L1639) 和 [`src/node_main_instance.cc`](../src/node_main_instance.cc#L33-L110) 连续跟踪。

### 4.2 Bootstrap 有“快照恢复”和“冷启动执行”两条路径

默认非交叉编译构建会生成 Node startup snapshot。构建期的 `node_mksnapshot` 预先执行只依赖确定性环境的 bootstrap，序列化可恢复的 V8 object graph 与 Node metadata，并生成配套的 builtin code cache；运行时通常恢复这些启动材料，而不是每次都完整重跑 `realm.js`/`node.js`。`--no-node-snapshot` 或特殊构建才走冷启动执行。这里的 code cache 是 builtin 编译缓存，不是把生产热点的最高层 JIT 机器码永久腌起来。

冷启动顺序大致是：

1. per-context `primordials`、DOMException、MessagePort 基础对象；
2. `internal/bootstrap/realm.js` 建立 binding loader 与 `BuiltinModule`；
3. `internal/bootstrap/node.js` 建立 `process`、global、Buffer、timer/task hooks；
4. Web globals 与 main/worker、是否拥有 process state 的 switches；
5. `pre_execution.js` 读取本次 argv、env、权限、IPC、preload 和 loader 配置；
6. `internal/main/*` 根据 CLI 模式进入文件、eval、REPL、test、watch 或 Worker 主程序。

这种拆分不是纯性能技巧。快照内容必须与构建机的 argv/env 无关；否则构建时状态会被“冻进”每个发行二进制。源码在 [`lib/internal/bootstrap/node.js`](../lib/internal/bootstrap/node.js#L7-L48) 直接记录了该约束，C++ 分派在 [`src/node_realm.cc`](../src/node_realm.cc#L195-L208) 和 [`src/node_realm.cc`](../src/node_realm.cc#L331-L372)。

### 4.3 普通主模块和 event loop 的先后关系

普通 CJS 主模块会在 `SpinEventLoopInternal()` 之前同步执行；ESM 路径先建立并链接模块图，求值过程 Promise 化，其 continuation 由 microtask 推进。要注意：一个 pending Promise 本身不自动让 libuv loop 永远存活，真正决定进程生命的仍包括 referenced handle/request 等实际工作。`StartExecution()` 先根据 worker、inspect、eval、test、watch、文件入口、REPL 等模式选择 `internal/main/*`，普通文件才进入 `run_main_module.js`。入口选择见 [`src/node.cc`](../src/node.cc#L288-L405)，CJS/ESM 分派见 [`lib/internal/modules/run_main.js`](../lib/internal/modules/run_main.js#L47-L164)。

### 4.4 Event loop 不只是一次 `uv_run()`

主循环会反复：

1. 调用 `uv_run(UV_RUN_DEFAULT)` 处理 libuv timer、poll、request completion 和 handle callback；
2. 让 NodePlatform drain 当前 Isolate 的 V8 foreground tasks；
3. 检查 loop 是否仍有 referenced handle/request；
4. loop 空时触发 `beforeExit`；若监听器新建 referenced work，则重新进入 loop；
5. 真正空闲后触发 `exit`，再按依赖反序清理 Environment、Worker、Realm、IsolateData、Isolate 与进程级 Platform。

实际实现只有几十行，位于 [`src/api/embed_helpers.cc`](../src/api/embed_helpers.cc#L23-L77)。它解释了为什么“libuv event loop”等同于整个 Node 调度器是不准确的：Node 还必须在 libuv、V8 foreground tasks、microtask、`process.nextTick()` 和退出事件之间建立交汇点。这里也要拆开两个名字很像的动作：`platform->DrainTasks(isolate)` 只负责 V8 Platform foreground tasks；Node 的 nextTick、V8 microtask 和 rejection checkpoint 则在 native→JS callback/task-queue 边界协调，不能都叫成一次 `DrainTasks()`。

## 5. 异步与并发模型

```mermaid
flowchart LR
  JS["某个 Environment 的 JS 线程：普通用户 JS callback run-to-completion"]
  API{"异步工作的性质"}
  NET["网络 socket / pipe：向平台 I/O 机制登记非阻塞操作"]
  FILE["FS、getaddrinfo、部分 crypto/压缩：提交可能阻塞的 native 工作"]
  CPU["CPU 密集用户 JavaScript：显式创建 Worker"]
  PROMISE["Promise reaction / queueMicrotask：当前 Context 关联的 V8 MicrotaskQueue"]
  TICK["process.nextTick：当前 Environment 的 Node queue"]

  POLL["OS I/O 通知：epoll/kqueue readiness；IOCP completion"]
  POOL["libuv 全局 worker pool：默认 4，可由 UV_THREADPOOL_SIZE 调整"]
  WORKER["Worker OS 线程：独立 Isolate、heap、GC、uv_loop 与 module cache"]
  LOOP["所属 event loop：接收 I/O / worker completion"]
  RUN["在所属 JS 线程运行用户 callback / Promise continuation"]
  CHECK["callback 边界 checkpoint：Node nextTick、V8 microtasks 与 rejection"]

  JS --> API
  API -->|非阻塞 I/O| NET --> POLL --> LOOP
  API -->|阻塞或 CPU 型 native work| FILE --> POOL --> LOOP
  API -->|并行 JS| CPU --> WORKER -->|MessagePort / structured clone| LOOP
  API -->|Promise / queueMicrotask| PROMISE --> CHECK
  API -->|Node 特有队列| TICK --> CHECK
  LOOP --> RUN --> CHECK --> JS

  V8BG["V8 Platform background threads：处理部分可并发 GC/编译等引擎工作"]
  V8BG -.需要进入 Isolate 时把 foreground task 投回.-> LOOP
```

可独立复用的图源位于 [`async-concurrency.mmd`](async-concurrency.mmd)。

### 5.1 四种“后台工作”不能混为一谈

| 机制 | 是否执行用户 JS | 是否有独立 JS heap | 典型工作 | 完成如何回到用户代码 |
| --- | --- | --- | --- | --- |
| OS I/O 通知机制 | 否 | 否 | Unix socket readiness；Windows I/O completion | 事件回到所属 `uv_loop_t`，再在该线程运行 JS callback；通知机制本身不执行 JS。 |
| libuv 全局 worker pool | 否 | 否 | 文件系统、`getaddrinfo/getnameinfo`、`uv_queue_work`，以及 Node 提交的部分 crypto/压缩任务 | native completion 被投递回提交请求的 loop，再进入 JS callback scope。 |
| `worker_threads` Worker | 是 | 是 | CPU 密集 JS、独立模块图和事件循环 | 通过 MessagePort、structured clone、transfer 或显式共享内存通信。 |
| V8 Platform background threads | 通常否 | 属于引擎内部 | 部分可并发 GC、编译和其他 VM 后台任务 | 需要进入目标 Isolate 的 foreground task 回投到其所属 loop。 |

libuv worker pool 是**进程全局共享**的，不是每个 Worker 各有四条线程；默认大小为 4，可在启动时通过 `UV_THREADPOOL_SIZE` 调整。增加它可能提高特定阻塞 native workload 的吞吐，但也增加内存和调度竞争，不会让 CPU 密集用户 JS 自动并行。证据见 [`deps/uv/docs/src/threadpool.rst`](../deps/uv/docs/src/threadpool.rst#L7-L30)。

### 5.2 网络 I/O 与文件 I/O 的关键差异

网络 socket 由平台 I/O 机制通知：Unix 上 epoll/kqueue 主要报告 readiness，Windows IOCP 更偏 completion；libuv 把这些差异封装成一致的 loop/callback 模型，因此网络等待通常不需要占用一条 worker thread。跨平台文件系统接口缺少统一、可靠的同类异步机制，libuv 因而把许多文件操作放到全局线程池。设计依据见 [`deps/uv/docs/src/design.rst`](../deps/uv/docs/src/design.rst#L65-L78)、[`deps/uv/docs/src/design.rst`](../deps/uv/docs/src/design.rst#L139-L162)。

### 5.3 `fs.readFile()` 的完整因果链

以 `fs.readFile()` 为例：

1. 公共 API 在 [`lib/fs.js`](../lib/fs.js#L387-L410) 校验参数、处理 AbortSignal/VFS hook，并建立 `ReadFileContext`。
2. `lib/fs.js` 从 `internalBinding('fs')` 获取 `FSReqCallback` 和 native 方法；`internalBinding()` 在当前 Realm 缓存导出。
3. C++ `fs` binding 由 [`src/node_file.cc`](../src/node_file.cc#L4341-L4343) 注册；异步分支创建 `ReqWrap` 并调用 `uv_fs_*`。
4. libuv 在线程池完成阻塞文件调用，把 completion 投回原来的 loop。
5. `AsyncWrap::MakeCallback()` 恢复 async id/trigger id 和 AsyncLocalStorage 上下文，调用 JS completion。
6. 最外层 native→JS callback 返回时，Node 执行 Promise microtask、`process.nextTick()` 和 rejection 检查。

这条链说明“异步 API 在线程池执行 JavaScript callback”是错误模型：线程池只做 native 工作，用户 callback 仍回到拥有该 Environment 的 JS 线程。

## 6. 三套模块加载器与两类 native 扩展边界

Node.js 不是只有一个 `require()` loader，而是至少有三套职责不同的装载平面。

| 装载平面 | 载荷 | 为什么独立 | 稳定性 |
| --- | --- | --- | --- |
| `BuiltinModule` | `lib/**/*.js` 与选定的内嵌依赖 JS | 内核必须在文件系统 API、用户 hooks 和用户模块出现之前可信启动，且避免启动时磁盘依赖。 | 受契约保护的是 `node:fs` 等公开核心模块的行为，不是内部 `BuiltinModule` 类或全部 builtin source；`lib/internal` 不保证稳定。 |
| CJS loader | 用户 `.cjs`/CommonJS、JSON、`.node` 等 | 保留同步 `require()`、`require.cache`、扩展搜索和历史 monkey patch 兼容。 | 公开行为高度兼容；内部实现可演进。 |
| ESM loader | URL 化的 ESM/WasM/TS 等模块图 | 需要 resolve/load/translate、link/evaluate、TLA 与同步/异步 customization hooks。 | 公开 API 按稳定等级；部分 hooks 仍可能演进。 |

`internal/bootstrap/realm.js` 先建立：

- `process.binding()`：用户可触达、但仅供 Node 内部使用且已运行时弃用的遗留入口；能调用不等于受支持；
- `process._linkedBinding()`：Embedder 静态链接额外 binding 的入口；
- `internalBinding()`：core-private 的 C++ binding loader；
- `BuiltinModule`：内嵌核心 JS 的最小模块系统。

`internalBinding()` 的 JS 缓存位于 [`lib/internal/bootstrap/realm.js`](../lib/internal/bootstrap/realm.js#L185-L208)，C++ 端在 [`src/node_binding.cc`](../src/node_binding.cc#L585-L663) 从内部注册表找到 module、创建 exports，并在当前 Realm/Context 初始化。

### 为什么 core JS 使用 `primordials`

用户可以修改 `Array.prototype`、`Promise` 等全局 intrinsic。若内核在处理安全、I/O 或错误路径时直接调用可被用户 monkey patch 的方法，用户代码就能改变内核运行语义。per-context bootstrap 因而提前保存安全引用和 `SafeMap`/`SafeSet` 等 primordials，内部模块通过闭包拿到它们。这是“宿主代码与 guest code 共享同一语言环境”所必需的防御边界，而不仅是风格约定。

## 7. 关键依赖及用途

当前源码快照中，Node 为 `27.0.0-pre`、V8 为 `14.6.202.34`、libuv 为 `1.52.1`、OpenSSL 为 `3.5.7`、npm 为 `11.18.0`、Undici 为 `8.7.0`。这些是仓库内容版本，不证明本机已安装或正在运行相同构建。

| 依赖或子系统 | 在 Node 架构中的用途 | 典型上层入口 |
| --- | --- | --- |
| V8 | ECMAScript/WebAssembly、JIT、GC、Isolate/Context、Promise microtask、Inspector 基础 | 几乎全部 JS；`vm`、`v8`、ESM ModuleWrap |
| libuv | loop、timer、TCP/UDP/pipe/TTY、process、signal、FS、系统 DNS、线程抽象和全局 pool | `net`、`dgram`、`fs`、`child_process`、`timers`、Worker thread primitive |
| OpenSSL + ncrypto | TLS、证书、随机数、hash/cipher/signature、Web Crypto 的 native 能力 | `tls`、`https`、`crypto`、`globalThis.crypto` |
| llhttp | HTTP/1 请求/响应解析 | `http` / `https` server 和 client |
| nghttp2 | HTTP/2 framing、session 与 stream | `http2` |
| c-ares | channel-based DNS query；与走系统 `getaddrinfo` 的 `dns.lookup()` 路径不同 | `dns.resolve*()` |
| Undici | Fetch、Request/Response/Headers、WebSocket/EventSource 等 WHATWG 网络 API | `fetch` 与相关 global，按需加载 |
| ICU + simdutf | Intl、Unicode、编码和高效 UTF 转换 | `Intl`、`TextEncoder`/`TextDecoder`、字符串/Buffer 边界 |
| Ada | WHATWG URL 解析 | `URL`、`url` binding |
| zlib / Brotli / zstd | 同步与异步压缩；异步路径可使用线程池 | `zlib`、HTTP content encoding |
| Amaro | TypeScript type stripping/transformation | `--strip-types` 与 TS 模块路径 |
| SQLite | 内建 SQLite 能力，是否开放受版本/实验开关约束 | `node:sqlite` |
| npm / Corepack | 随发行包交付的包管理工具，不是 Node 主进程启动链的核心库 | 安装后的 `npm`/Corepack 命令 |
| postject / LIEF | Single Executable Application 的资源注入/编辑能力 | SEA 构建路径 |
| googletest | C++ 测试依赖，不属于生产运行时主路径 | `test/cctest` |

`deps/` 很大，但不能由“位于 deps”推导出“全部链接并在每次运行中初始化”。有些依赖被静态链接，有些 JS 被 `js2c` 嵌入，有些只用于构建、测试或随发行包交付；部分原生库还可以由下游构建 externalize。

### 7.1 公共核心子系统如何跨层实现

| 子系统 | JS 层主要负责 | Native/依赖层主要负责 | 关键运行特征 |
| --- | --- | --- | --- |
| timers / immediate | timer 列表、优先级、重复调度和回调语义 | 每个 Environment 的少量 `uv_timer_t`/check handle | 不是每个 timer 一条线程；JS 用最近到期时间重设 native timer。 |
| streams | readable/writable 状态机、pipe、Transform、`highWaterMark` 与 `drain` 背压 | socket/file/crypto/zlib 等具体 source/sink | 背压限制速率差转化为的内存增长，但不会把阻塞计算自动搬离 loop。 |
| net / dgram / pipe | Socket/Server API、连接状态、错误和事件语义 | TCP/UDP/PipeWrap、libuv handle 和 OS I/O 通知 | Unix 多为 readiness、Windows 可为 completion；通常不为每个连接占一个线程。 |
| HTTP/1 | request/response、Agent、socket 生命周期与 header 语义 | llhttp 同步解析收到的 Buffer | parser 与部分 JS 协议逻辑仍运行在 loop 线程，大包/重回调会占用 loop。 |
| HTTP/2 | JS session/stream API 与 socket 接管 | nghttp2 frame、HPACK、session 状态机 | 多路复用不等于多个 JS 线程。 |
| Fetch / WebSocket | WHATWG API 与客户端策略 | 内嵌 Undici JS，最终复用 Node 网络/TLS 能力 | 与传统 `node:http` 是不同的上层客户端栈，并按需加载。 |
| DNS | `dns` API、参数/结果语义 | `dns.lookup()` 使用系统 getaddrinfo/thread pool；`dns.resolve*()` 使用 c-ares socket/loop | 两类 API 共享“DNS”名称，却具有不同调度和缓存/解析语义。 |
| TLS / crypto | JS API、Key/Object、Web Crypto promise 语义 | OpenSSL/ncrypto；部分 async CryptoJob 进入 pool | TLS stream 上的部分计算仍在 loop 线程；只有显式异步 job 才离开。 |
| diagnostics | channel/hook API、上下文和用户观察面 | AsyncWrap、trace events、V8 Inspector、性能里程碑 | `diagnostics_channel.publish()` 本身是同步通知；观测代码也可能影响被观测路径。 |

这进一步解释了“JS 策略、native 机制”的边界不是按模块机械切半：streams 大多是 JS 状态机，网络 handle 和文件 syscall 必须 native，HTTP/TLS 则跨 JS、C++ 和专用库共同完成。

## 8. 构建期架构

```mermaid
flowchart LR
  CONFIG["configure.py：探测平台、编译器、CPU、依赖和功能开关"]
  GYPI["config.gypi：本次构建的事实配置"]
  GYP["node.gyp + common.gypi + node.gypi：官方二进制的权威构建图"]
  BACKEND["GYP 后端：Make / Ninja / MSBuild / Xcode"]

  LIBJS["lib/**/*.js：公共 API、internal、bootstrap、main"]
  SHAREJS["deps/undici、Amaro 与选定 V8 工具 JS"]
  JS2C["node_js2c：把 JS 与 config.gypi 转成 C++ 嵌入数据"]
  EMBED["node_javascript.cc：BuiltinModule 可读取的内嵌源码"]

  CPPSRC["src/**/*.cc：runtime、binding、module、worker、diagnostics"]
  NATIVEDEPS["deps/：V8、libuv、OpenSSL、llhttp、nghttp2、ICU、压缩库等"]
  NODEBASE["node_base：Node C++ 核心 + 内嵌 JS"]
  LIBNODE["libnode：Node 可执行文件和 embedder 共用的运行时库"]

  MKSNAPSHOT["node_mksnapshot：执行 snapshot-safe bootstrap 并生成 code cache"]
  SNAPSHOT["node_snapshot.cc：预初始化 V8 heap；可按配置禁用或替换"]
  MAINCC["src/node_main.cc：极薄的平台 main / wmain 入口"]
  BIN["node 可执行文件：libnode + main + snapshot + 选定依赖"]

  TEST["test/ + tools/test.py：JS 行为、C++、addons、Node-API、WPT 与平台测试"]
  BENCH["benchmark/：组件与跨模块性能测量，不自动等于端到端生产结论"]
  DOC["doc/ + tools/doc：API 文档、弃用与兼容契约"]
  GN["BUILD.gn：非官方/辅助构建入口；构建系统改动应回写 GYP"]

  CONFIG --> GYPI --> GYP --> BACKEND
  LIBJS --> JS2C
  SHAREJS --> JS2C
  GYPI --> JS2C --> EMBED --> NODEBASE
  CPPSRC --> NODEBASE
  NATIVEDEPS --> NODEBASE
  NODEBASE --> LIBNODE
  NODEBASE --> MKSNAPSHOT --> SNAPSHOT --> LIBNODE
  LIBNODE --> BIN
  MAINCC --> BIN
  BACKEND --> JS2C
  BACKEND --> NODEBASE
  BACKEND --> MKSNAPSHOT
  TEST -->|验证| BIN
  BENCH -->|测量| BIN
  DOC -->|定义公开行为| LIBJS
  GN -.辅助路径.-> BIN
```

可独立复用的图源位于 [`build-architecture.mmd`](build-architecture.mmd)。

### 8.1 权威构建链

标准链路是：

`./configure` → `configure.py` → `config.gypi/config.mk` → GYP → Make/Ninja/MSBuild → `node.gyp` target graph → `node`

`node.gyp` 定义的核心 target 关系是 `node executable → libnode → node_base`。`node` target 本身只有很薄的 `src/node_main.cc`，大部分源码进入 `node_base`/`libnode`。[`BUILD.gn`](../BUILD.gn#L1-L13) 明确说明 GN 不是官方二进制的构建系统；只修改 GN 即使本地成功，也不能证明官方发布路径包含变更。

### 8.2 为什么把 JS 编进 C++ 二进制

`configure.py` 枚举 `lib/**/*.js`，`node_js2c` 再把这些文件、选定依赖 JS 和 `config.gypi` 生成 `node_javascript.cc`。因此核心模块：

- 不依赖安装目录里的散落 JS 文件；
- 在 `fs` 和用户模块 loader 尚未可用时就能启动；
- 与 native binary 和构建配置保持原子版本；
- 可进一步生成 startup snapshot/code cache，减少每次进程启动的初始化成本。

代价是修改 `lib/` 后默认需要重新生成/链接。开发模式 `--node-builtin-modules-path` 可从磁盘读取 builtins，以换取快速迭代，但它会关闭 Node snapshot/code cache，不能代表正式自包含二进制的启动路径。

关键 build action 位于 [`node.gyp`](../node.gyp#L1073-L1105)，snapshot action 位于 [`node.gyp`](../node.gyp#L1146-L1196)。

## 9. 仓库目录地图

| 路径 | 责任 | 阅读建议 |
| --- | --- | --- |
| `src/` | C++ runtime、binding、V8/libuv glue、Worker、Inspector、Node-API/Embedder public headers | 从 `node_main.cc`、`node.cc`、`node_main_instance.cc`、`api/environment.cc` 开始，不要按文件名全量扫。 |
| `lib/` | 公共 JS core API、`lib/internal`、bootstrap、main modes、CJS/ESM loader | 用一个 API 做纵向追踪，例如 `fs.js → internalBinding('fs') → node_file.cc`。 |
| `deps/` | vendored JS/native 依赖、构建/测试依赖和随发行包交付工具 | 先看 `node.gyp/node.gypi/configure.py` 是否真的把它接入目标路径。 |
| `tools/` | GYP、js2c、snapshot、test runner、lint、doc、依赖更新和 release 工具 | 构建/生成问题优先从对应 tool 和 `node.gyp` action 双向查。 |
| `test/` | JS 行为、C++、addon、Node-API、WPT、message、TTY、sequential/parallel、已知问题和负载测试 | `test/README.md` 解释每类测试能证明什么。 |
| `benchmark/` | 按子系统组织的 benchmark、compare runner 与统计/可视化工具 | 结果只支持实际执行的路径；组件基准不能外推生产端到端结论。 |
| `doc/api/` | 公共 API、稳定等级、弃用和兼容契约的源码 | 修改公开行为时，代码、测试和 API 文档必须一致。 |
| `doc/contributing/` | 构建、测试、发布、维护依赖等项目工作流 | 判断“官方路径”与“辅助路径”的主要依据。 |
| `typings/` + `tsconfig.json` | core 内部静态类型辅助，`noEmit` | 不是用户侧 `@types/node`，也不是 runtime 构建产物。 |
| `.github/` | CI、CODEOWNERS、automation 与仓库协作配置 | CI 证明的仍是相应 job/平台实际覆盖范围。 |

根级没有 `package.json` 是一个重要分类信号：尽管 `deps/npm`、`deps/undici` 等子树各有 package metadata，Node core 的顶层构建不是 npm scripts 驱动的应用工程。

## 10. 核心设计思想：因果与反事实

| 设计 | 为什么存在 | 如果反过来设计会怎样 |
| --- | --- | --- |
| Node 做编排，V8/libuv 各司其职 | 语言执行、OS 异步和宿主 API 是三类独立问题；组合比重写引擎或每个平台各写 runtime 更可维护。 | 只有 V8 没有 I/O/进程/模块；只有 libuv 没有 JS；每个平台直接写 I/O 会让语义和 bug 修复分裂。 |
| JS 负责策略，C++ 负责机制 | 模块、stream、兼容与校验逻辑需要快速、安全演进；引擎/OS 边界、资源句柄和热路径必须 native。 | 全部 C++ 会让 API 迭代成本高；全部 JS 又无法可靠访问 V8 与 OS 原语。 |
| 普通用户 JS 在所属 Isolate 线程 run-to-completion，parallelism 显式化 | 普通 GC-managed object graph 不跨 Isolate 直接共享，callback 避免任意抢占；并行需求交给独立 Worker、消息/transfer 或显式 SharedArrayBuffer + Atomics。V8 仍可受控地用后台线程做部分 GC/编译。 | 任意线程同时进入同一 JS object graph 会引入数据竞争、锁开销和不可预测 GC；完全不提供 Worker 又无法利用多核跑 CPU 密集 JS。 |
| 一个 Environment 绑定一个 loop/Isolate/principal Realm | 把 Node 实例状态、VM 状态和 I/O 生命周期形成可清理边界，也支持 Worker 与 Embedder 多实例。 | 全局单例承载全部状态会让 Worker 隔离、独立退出、资源追踪与嵌入场景失效。 |
| Builtin/CJS/ESM loader 分离 | 内核要在 `fs` 和用户 hooks 之前启动；CJS 与 ESM 又具有不同解析、缓存和求值语义。 | 用 CJS loader 启动自身会出现 loader/`fs` 循环依赖；强行统一 CJS/ESM 会破坏同步 `require()` 或 TLA/hooks。 |
| 内嵌 JS + startup snapshot | 二进制自包含、构建配置与 core JS 原子一致，并把确定性初始化移出每次启动。 | 从磁盘加载会增加部署路径、I/O、篡改和版本错配问题；每次完整 bootstrap 会增加启动成本。 |
| primordials 隔离内核 | core 与用户共享一台 JS VM，必须防止用户 monkey patch 改写内核依赖的 intrinsic。 | 内部校验、容器和错误路径会被用户修改，出现安全与正确性问题。 |
| BaseObject/AsyncWrap 显式协调生命周期与因果 | JS GC、OS handle、native request 和 AsyncLocalStorage 的生命规则不同。 | 仅靠 GC 会过早释放 native 资源；仅靠 native 引用会泄漏；不记录 trigger id 会丢失异步上下文。 |
| 分阶段清理 | Worker、handle、Realm、Environment、Isolate 与 Platform 有严格反向依赖。 | 先销毁 Isolate 再关闭 handle，迟到 completion 会触发 use-after-free；省略 drain/cleanup 会泄漏资源。 |
| 稳定层与内部层分离 | 公共生态需要兼容，core 内部又必须持续重构；Node-API 还要隔离 V8 ABI 变化。 | 所有内部都承诺兼容会冻结实现；完全不设稳定层会迫使每个 native addon 随 Node major 重编译。 |

## 11. 兼容与扩展面

| 扩展面 | 能力 | 兼容边界 |
| --- | --- | --- |
| 公共 JS API | `node:fs`、`node:http`、Web globals 等 | 以 `doc/api` 稳定等级、弃用政策和 SemVer 为契约；Experimental 需要单独审计。 |
| CJS/ESM customization hooks | 自定义 resolve/load 与模块策略 | 同步和异步 hooks 的稳定等级可能不同；还要考虑 loader worker 与主线程边界。 |
| Node-API | 用 C ABI 构建 `.node` addon | 推荐；设计目标是跨 Node major 的 ABI 稳定，当前源码支持 Node-API 1..10。混用 V8/Node C++ API 会削弱保证。 |
| 直接 V8/Node C++ addon | 访问底层 V8、libuv、Node C++ interface | 受 `NODE_MODULE_VERSION` 约束，当前为 147；不匹配时 `process.dlopen` 会拒绝加载。 |
| C++ Embedder API | 在其他 C++ 应用中创建 Node Environment | 能力强，但允许在 semver-major 中发生无预警 breaking change。 |
| linked JS/native modules | 构建时把额外 JS 或 binding 静态链接进宿主 | 属于定制构建/嵌入边界，不是一般 npm package 兼容面。 |
| `lib/internal` / `internalBinding` | core 维护和调试 | 不向用户承诺兼容；`--expose-internals` 也仅供维护调试。 |

Node-API 的设计价值可以用反事实说明：直接 addon 的符号和对象布局依赖 V8/Node C++ ABI，V8 升级可能迫使 addon 重编译；Node-API 用不透明句柄和 C 函数表把 addon 与具体 JS 引擎隔开。源码说明见 [`doc/api/n-api.md`](../doc/api/n-api.md#L7-L20)，直接 addon 的 ABI 检查见 [`src/node_binding.cc`](../src/node_binding.cc#L515-L560)。

## 12. 测试、基准、文档与发布各证明什么

先建立一个不会被漂亮数字骗走的公式：**证据 = 方法 × 实际执行路径 × 环境与持续时间**。`test` 是带断言的验证方法，`benchmark` 是计时/计数方法；它们都可以只测 component，也可能覆盖 end-to-end。秒表负责计时，不负责替你宣布生产胜利。

| 证据类型 | 主要入口 | 能证明 | 不能自动证明 |
| --- | --- | --- | --- |
| 组件/行为测试 | `tools/test.py`，`test/parallel`、`sequential`、`cctest`、addons、Node-API、WPT 等 | 指定代码路径和平台组合的功能/回归行为 | 所有平台、长时运行、生产负载和完整发布包行为 |
| 负载/压力测试 | `test/pummel` 等 | 特定场景在更大压力下的正确性 | 通用性能 SLO 或内存稳态 |
| Benchmark | `benchmark/run.js`、`compare.js` | 实际 benchmark 路径的吞吐/延迟差异 | 未执行的 end-to-end 路径、soak 或 production 收益 |
| API 文档测试 | `doc/api` + doc tools | 文档结构、链接和部分示例/契约一致性 | 实现行为本身完全正确 |
| CI | `.github/workflows` 与外部 Jenkins | 相应 job 覆盖的平台、配置与测试集合 | 未覆盖的 external/shared deps、发行环境或长期稳态 |
| Release tooling | Make targets、`tools/release.sh` 与外部发布基础设施 | 产物生成、校验和、签名和提升步骤 | 仅靠仓库本地脚本不能证明外部 Jenkins/nodejs.org 发布全链成功 |

路径和环境范围应单独标注：

| 标签 | 人话解释 | 结论边界 |
| --- | --- | --- |
| `proxy` | 用替代指标或缩小模型近似真实问题，像先量灶台温度猜出餐速度。 | 只能证明代理指标；必须额外论证它与目标结果的因果关系。 |
| `component` | 只穿过一个子系统或清晰局部边界，像只测榨汁机。 | 不能自动覆盖调用方、排队、网络、持久化和组合调度。 |
| `end_to_end` | 从真实外部入口穿过主要层次直到可观察结果，像完整点单到上菜。 | 路径完整不等于环境真实，也不等于长时间稳定。 |
| `soak` | 持续较长时间运行，观察泄漏、积压、漂移和偶发失败，像连续营业数天。 | 能支持稳态结论，但仍取决于负载模型和环境。 |
| `production` | 在真实部署与真实流量下观测，终于到了真门店。 | 现实相关性最高，但变量最多；相关变化不自动等于单一代码变更导致。 |

`test/README.md` 把测试按用途分类，例如 `parallel`、`sequential`、`cctest`、addons、Node-API、internet、pummel 和 known issues；`benchmark/README.md` 则把性能测量单独组织。这种分离本身表达了项目理念：正确性证据和性能证据不可互相替代。

## 13. 推荐阅读路径

如果目标是快速建立可调试的心智模型，建议按因果链而不是目录字母序阅读：

1. **进程入口**：`src/node_main.cc` → `src/node.cc::StartInternal()`。
2. **实例创建**：`src/node_main_instance.cc` → `src/api/environment.cc` → `src/env.cc`。
3. **bootstrap**：`lib/internal/bootstrap/realm.js` → `node.js` → `internal/process/pre_execution.js`。
4. **主模块**：`lib/internal/main/run_main_module.js` → `internal/modules/run_main.js` → CJS/ESM loader。
5. **一个纵向 API**：`lib/fs.js` → `internalBinding('fs')` → `src/node_binding.cc` → `src/node_file.cc` → libuv。
6. **loop 与 callback**：`src/api/embed_helpers.cc`、`src/api/callback.cc`、`lib/internal/process/task_queues.js`。
7. **并行**：`lib/internal/worker.js` → `src/node_worker.cc` → MessagePort 实现。
8. **构建逆向**：`configure.py` → `node.gyp` → `tools/js2c.cc` → `tools/snapshot/`。
9. **契约与验证**：对应 `doc/api`、`test`、`benchmark` 和 CI job。

## 14. 关键源码证据索引

| 结论 | 当前提交中的证据 |
| --- | --- |
| Native main 只汇入 `node::Start` | `src/node_main.cc:34-98` |
| `StartInternal` 完成进程初始化、snapshot 与主实例创建 | `src/node.cc:1564-1639` |
| 主实例创建 Environment、加载 JS 并进入 loop | `src/node_main_instance.cc:33-110` |
| Environment = loop + Isolate + principal Realm | `src/README.md:307-336` |
| 冷启动 bootstrap 顺序 | `src/node_realm.cc:195-208,331-372` |
| 内建 JS、binding loader 与用户 loader 的区分 | `lib/internal/bootstrap/realm.js:1-39,185-238` |
| `StartExecution` 的 CLI 模式分派 | `src/node.cc:288-405` |
| CJS/ESM 主入口选择 | `lib/internal/modules/run_main.js:47-164` |
| event loop、V8 task 与 `beforeExit` 的交汇 | `src/api/embed_helpers.cc:23-77` |
| nextTick/microtask/rejection 队列 | `lib/internal/process/task_queues.js:49-109`、`src/node_task_queue.cc:129-194` |
| Worker 独立 loop/Isolate/Environment/OS thread | `src/node_worker.cc:159-220,296-435,727-773` |
| libuv 全局线程池与回投 loop 语义 | `deps/uv/docs/src/threadpool.rst:7-49` |
| 内建 JS 生成与嵌入 | `node.gyp:1073-1105` |
| snapshot 生成并进入 libnode | `node.gyp:1146-1196,1727-1813` |
| GN 不是官方发布构建图 | `BUILD.gn:1-13` |
| Node-API 的 ABI 稳定目标 | `doc/api/n-api.md:7-20` |
| 直接 addon 的 ABI 版本拒绝逻辑 | `src/node_binding.cc:515-560` |

## 15. 证据边界

本文已做到：

- 以当前 HEAD 的源码和仓库文档交叉验证构建、启动、module、binding、async、Worker、cleanup 与兼容边界；
- 将 build-time、runtime、线程/Isolate、public/internal 和 component/end-to-end 证据范围分开；
- 提供可维护 Mermaid 图源，而不是不可追踪的静态截图。

本文没有声称：

- 当前 HEAD 已在本机成功编译、安装或激活；仓库中没有 `out/` 产物；
- 某个具体 Node 应用的生产吞吐、延迟、内存或 soak 状态；
- 所有 `deps/` 在所有构建配置中都被静态链接或启用；
- 仅凭组件 benchmark 就能推出端到端或生产性能结论。

因此，本交付状态是**源码架构已取证并文档化**，不是 runtime/system/production verification。
