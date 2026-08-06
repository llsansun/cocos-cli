# Cocos CLI 项目知识速查

> 自动生成，基于项目探索和实际运行体验。更新于 2026-08-04。

---

## 一、环境要求

| 依赖 | 版本 | 说明 |
|---|---|---|
| Node.js | **22.17.0** | `.nvmrc` 指定，`WebSocket` 全局对象需要 Node 22+ |
| Git | 最新版 | 克隆仓库和引擎代码 |
| npm | 随 Node 附带 | 版本管理使用 nvm |

### 常见问题

| 问题 | 原因 | 解决 |
|---|---|---|
| `WebSocket is not defined` | Node v20 无全局 WebSocket | `nvm use 22.17.0` |
| `NODE_MODULE_VERSION 不匹配` | native 模块编译版本不对 | `npm rebuild` |
| `zsh: permission denied: cocos` | `dist/cli.js` 无执行权限 | `chmod +x dist/cli.js`（已加进 build 脚本） |
| GitHub clone 超时 | 国内网络 | 配置 git 代理 `git config --global http.proxy http://127.0.0.1:7890` |

---

## 二、安装与构建

```bash
# 1. 安装依赖
npm install -g node-gyp
npm run init      # 克隆 engine + external 仓库
npm install

# 2. 构建并链接
npm run build
npm link

# 切 Node 版本后需要：
nvm use 22.17.0
npm rebuild       # 重编 native 模块
npm run build
npm link
```

---

## 三、项目结构

```
cocos-cli/
├── src/              # 主源码
│   ├── cli.ts            # CLI 入口 (Commander.js)
│   ├── index.ts          # VS Code 插件入口
│   ├── commands/         # 命令层 (build/create/preview/...)
│   ├── api/              # API 层 (CocosAPI)
│   ├── core/             # 核心调度 + 子系统
│   │   ├── launcher.ts       # 调度器 (init→import→业务→close)
│   │   ├── engine/           # 引擎配置与生命周期
│   │   ├── builder/          # 多平台构建 (12+ 平台)
│   │   ├── assets/           # 资源数据库
│   │   ├── scene/            # 场景编辑器 Runtime
│   │   ├── scripting/        # 脚本编译与热重载
│   │   ├── configuration/    # 项目配置管理
│   │   ├── preview/          # 浏览器 & 场景预览
│   │   └── base/             # 日志/Sentry/i18n/工具
│   ├── server/          # Express + Socket.IO 服务
│   ├── mcp/             # MCP 协议集成
│   └── display/         # CLI 交互 UI
├── packages/
│   ├── engine/          # Cocos 引擎
│   │   ├── cocos/           # TS 引擎源码
│   │   ├── native/          # C++ 引擎源码
│   │   └── editor/          # 编辑器资源
│   ├── cc-module/       # 引擎前置模块 (node_modules/cc)
│   ├── cocos-cli-types/ # API 类型定义包
│   └── engine-compiler/ # 引擎编译器
├── workflow/           # 构建脚本 (init/build/generate/release)
├── docs/               # 中英文文档 + 开发者文档
├── e2e/                # 端到端测试
├── dist/               # 编译产物
└── static/             # 静态资源
```

### 架构图

```
CLI (cli.ts) / VS Code Extension (index.ts)
        │
        ▼
   Commands (commands/)  ←→  CocosAPI (api/)
        │                      │
        ▼                      ▼
   Core Orchestrator (core/launcher.ts)
        │
        ├── server/         HTTP + WebSocket
        ├── engine/         引擎配置&生命周期
        ├── assets/         资源DB & 导入管线
        ├── builder/        多平台构建
        ├── scene/          场景编辑 Worker+RPC
        ├── scripting/      TS编译&热重载
        ├── configuration/  项目配置
        ├── preview/        浏览器&场景预览
        └── mcp/            AI工具集成
```

---

## 四、引擎双层层架构

```
TS 引擎层 (packages/engine/cocos/)
        │ 游戏逻辑
        ↓
    ┌───┴──────────────────────┐
    │                          │
原生平台 (Android/iOS/Mac/...)   Web 平台
    │                          │
JsbBridge (V8 绑定)           WebAssembly (Emscripten)
    │                          │
C++ 引擎 (native/)            C++ → WASM
静态库/动态库                  WebAssembly.instantiate
```

### 构建 TS 引擎

```bash
cd packages/engine
npm run build:dev              # 开发版（未压缩）
npm run build:min              # 压缩版
# 或在项目根：
npm run compiler:engine -- --force   # 完整编译
```

### 构建 C++ Native

```bash
# 先下载外部依赖
cd packages/engine
npm run update:native-external

# 各平台构建
./native/utils/generate_compile_commands_mac.sh      # macOS
./native/utils/generate_compile_commands_ios.sh      # iOS
./native/utils/generate_compile_commands_android.sh  # Android
./native/utils/generate_compile_commands_windows.sh  # Windows
```

---

## 五、CLI 命令

| 命令 | 说明 |
|---|---|
| `cocos create -j <path> -t (2d\|3d)` | 创建项目 |
| `cocos build -j <path> -p <platform>` | 构建项目 |
| `cocos preview -j <path> [-p port] [--scene-editor]` | 预览项目 |
| `cocos make <platform> <dest>` | 打包 (APK/IPA) |
| `cocos run <platform> <dest>` | 运行到设备 |
| `cocos upload <platform> <dest>` | 上传构建产物 |
| `cocos start-mcp-server --project <path>` | 启动 MCP 服务 |

### npm scripts

| 命令 | 说明 |
|---|---|
| `npm run init` | 初始化（克隆引擎仓库） |
| `npm run build` | 完整构建 CLI |
| `npm run build:watch` | 监听模式 |
| `npm run test` | 单元测试 |
| `npm run test:e2e` | E2E 测试 |
| `npm run compiler:engine -- --force` | 编译引擎 |

---

## 六、执行流程

### Launcher 生命周期

```
构造 → init() → import() → 业务操作 → close()
```

| 阶段 | 操作 |
|---|---|
| `init()` | 日志系统 → 项目配置 → 项目打开 → 引擎初始化 |
| `import()` | 脚本编译初始化 → 资源DB启动（扫描导入 assets） |
| `build()` | builder 平台插件初始化 → 参数校验 → BuildTask.run() |
| `preview()` | Server 启动 → 注册路由 → 打开浏览器 |
| `close()` | 预览释放 → 停服 → 停场景进程 → 关资源DB → 关脚本 → 保存项目 |

### Build 流程

```
createBuildTask(platform, options)
  └─ pluginManager.checkOptions()   校验 startScene/scenes 等
  └─ BuildTask.run()
       ├─ 资源编译（场景/预制体/贴图…）
       ├─ 脚本打包（TS → JS）
       ├─ 引擎裁剪合并
       └─ 输出到 build/<platform>/
```

> 空项目 build 报 `startScene("") validation failed` 是因为没有场景文件，需先复制模板场景到 `assets/`。

### Server 层

基于 Express + Socket.IO，提供：
- `/` — 静态文件托管
- `/api/*` — REST API
- `/scene-editor/` — 场景编辑器 UI
- `/build/*` — 构建产物

### `npm run build` 流程

```
1. build:clear            清空 dist/
2. prepare-dts.js         准备类型生成环境
3. tsc -b                 编译 src/ → dist/
4. build:static-web       打包静态前端资源
5. generate-schema.js     JSON Schema
6. generate:dts           生成 .d.ts 类型包
7. build:platform-views   打包平台视图
8. chmod +x dist/cli.js  可执行权限
```

---

## 七、CLI 与引擎交互机制

### 三层桥接架构

```
CLI (src/)  ──→  cc-module (packages/cc-module/)  ──→  Engine (packages/engine/)
```

### 引擎初始化链

```
Launcher.init()
  → initEngine(enginePath, projectPath)        // src/core/engine/index.ts
    → Engine.init(enginePath)                   // 扫描引擎目录
    → Engine.initEngine(info)                   // 核心初始化
      → import('cc/preload')                    // 加载 cc-module 的 preload
      → preload(options)                        // 设置 globalThis 环境
        → globalThis.CC_EDITOR = false           // 模拟浏览器环境
        → globalThis.nodeEnv = { require, WebSocket, fetch, ... }
        → require('web-adapter.js')              // DOM API 适配
        → EngineLoader.init()                    // 劫持 Node.js 模块系统
        → require('cc')                          // 加载引擎核心
      → cc.game.init(config)                     // 启动引擎运行时
```

### 模块劫持机制（关键）

`packages/cc-module/src/loader.ts` 中的 `EngineLoader` 劫持了 Node.js 的模块加载：
- 拦截 `Module._resolveFilename` 和 `Module._load`
- 当代码 `require('cc')` 或 `require('cc/editor/...')` 时
- 不走 `node_modules`，而是重定向到 `packages/engine/` 下的实际文件

这样 CLI 代码中写 `require('cc')` 就能直接拿到引擎对象，CLI 代码可直接调用引擎 API：
```typescript
const cc = require('cc');
cc.game.init(config);
cc.assetManager.loadAny(...);
cc.director.loadScene(...);
```

### Polyfill + Overwrite 补丁层

`cc-module` 通过 `polyfill/engine.ts` 和 `overwrite.ts` 对引擎行为做 CLI 环境适配：
- 替换文件 I/O（浏览器 fetch → Node.js fs）
- 替换网络层（浏览器 XMLHttpRequest → Node.js xhr2）
- 禁用渲染循环（CLI 不需要逐帧渲染）

---

## 八、MCP/AI 交互体系

### 完整架构

```
AI 客户端 (Cursor/VS Code)
    │
    │  POST /mcp (JSON-RPC)
    ▼
MCP Server (src/mcp/)
    │  StreamableHTTPServerTransport
    ▼
Tool Registry (@tool 装饰器)
    │  Zod 校验参数
    ▼
Tool Handler (具体方法)
    │  调用 CocosAPI / Launcher
    ▼
Engine Runtime (cc.*)
```

### Step 1: @tool 装饰器自动注册（编译期）

```typescript
// src/api/decorator/decorator.ts
const toolRegistry = new Map<string, { target: any; meta: ToolMetaData }>();
```

TypeScript 装饰器在**模块被 import 时立即执行**，把方法元信息写入全局 `toolRegistry`。

每个 `@tool` 装饰的方法自动注册：
- 方法名（如 `'builder-build'`）
- 参数 Zod schema（从 `@param` 装饰器提取）
- 返回类型 schema（从 `@result` 装饰器提取）
- 描述信息（从 `@description` 装饰器提取）

### Step 2: CocosAPI.create() 触发全量 import

```typescript
// src/api/index.ts
private async _init() {
    const { SceneApi } = await import('../api/scene/scene');
    const { EngineApi } = await import('../api/engine/engine');
    const { ProjectApi } = await import('../api/project/project');
    const { AssetsApi } = await import('../api/assets/assets');
    const { BuilderApi } = await import('../api/builder/builder');
    const { ConfigurationApi } = await import('../api/configuration/configuration');
    const { SystemApi } = await import('../api/system/system');
}
```

不需要手动扫描文件路径——装饰器副作用在 import 时自动执行，一次性收集所有工具。

### Step 3: registerDecoratorTools() 注册到 MCP Server

```typescript
// src/mcp/mcp.middleware.ts 第127行
private registerDecoratorTools() {
    Array.from(toolRegistry.entries()).forEach(([toolName, { target, meta }]) => {
        this.server.tool(toolName, meta.description, inputSchemaFields, async (args) => {
            const result = await this.callToolMethod(target, meta, methodArgs);
            return { content: [{ type: 'text', text: formattedResult }] };
        });
    });
}
```

### Step 4: 覆盖 tools/list 暴露给 AI

```typescript
// src/mcp/mcp.middleware.ts 第210行
this.server.server.setRequestHandler(ListToolsRequestScheme, async () => {
    return {
        tools: Array.from(toolRegistry.entries()).map(([toolName, { meta }]) => ({
            name: toolName,
            description: meta.description,
            inputSchema: geminiInputSchema,   // AI 看到参数定义
            outputSchema: geminiOutputSchema  // AI 看到返回格式
        }))
    };
});
```

### 请求处理链

```
AI 发送: POST /mcp
  { method: "tools/call", params: { name: "queryAssets", arguments: { type: "Scene" } } }
    │
    ▼
McpMiddleware.handleMcpRequest()
  → transport.handleRequest()
  → MCP SDK 路由到 tool handler
  → Zod 验证输入参数
  → callToolMethod() 调用实际方法
  → 方法内部调用 CocosAPI / 引擎 API
  → Zod 验证返回值
  → formatToolResult() 格式化
  → 返回 JSON-RPC 响应给 AI
```

### 全部 57 个工具分类

| 类别 | 工具数量 | 做什么 |
|------|---------|--------|
| 场景编辑 | 20 个 | 场景 CRUD、节点创建/删除/查询/更新、组件增删改查、Prefab 操作 |
| 资源管理 | 31 个 | 资源创建/导入/复制/删除/重命名/移动、材质查询/保存、动画蒙皮、序列化数据读写 |
| 构建打包 | 6 个 | 构建、生成 Make/Run/Upload、查询默认配置 |
| 文件编辑 | 4 个 | 文本插入/删除/替换/查询（操作脚本源码） |
| 系统 | 2 个 | 查询/清空日志 |
| 配置 | 1 个 | 配置重新迁移 |

### 资源暴露

MCP 通过 Resources 向 AI 暴露项目信息：
- `cli://docs/*` — 项目文档（README、SUMMARY 等）
- `cocos://assets/{ccType}` — 按类型查询项目资产（Scene、Prefab、Material 等）
- `cocos://docs/api` — 引擎 API 参考

---

## 九、场景编辑 RPC 机制

### 架构：双进程 + RPC

```
┌─────────────────────────────┐      ┌──────────────────────────────────┐
│      CLI 主进程 (Main)       │      │    场景 Worker 子进程 (Child)     │
│                              │      │                                  │
│  SceneApi.createByType()     │      │  NodeService.createByType()      │
│       │                      │      │       │                          │
│       ▼                      │      │       ▼                          │
│  Rpc.getInstance()           │      │  Service.Editor.getRootNode()    │
│  .request('Node','create..') │      │       │                          │
│       │                      │      │       ▼                          │
│       ▼                      │      │  new cc.Node()  ← 真实引擎对象   │
│  process.send(req)           │      │  node.setParent(...)             │
│       │    JSON 消息          │      │  node.setPosition(...)          │
│       ├─────────────────────►│      │       │                          │
│       │  { type:'request',   │      │       ▼                          │
│       │    module:'Node',    │      │  generateNodeDump() → JSON       │
│       │    method:'create..',│     │       │                          │
│       │    args:[params] }   │      │       ▼                          │
│       │                      │      │  process.send(res)               │
│       │  { type:'response',  │      │                                  │
│       │◄─────────────────────┤      │                                  │
│       │    result: INode }   │      │                                  │
│       ▼                      │      │                                  │
│  DumpConverter.toNode()      │      │                                  │
│  → 返回给 AI                  │      │                                  │
└─────────────────────────────┘      └──────────────────────────────────┘
```

### Step 1: 启动时 fork 子进程

```typescript
// src/core/scene/main-process/scene-worker.ts
this._process = fork(workerPath, args, {
    stdio: ['pipe', 'pipe', 'pipe', 'ipc']  // IPC 通道
});
```

### Step 2: 子进程注册 Node 服务（引擎侧）

```typescript
// src/core/scene/scene-process/service/core/decorator.ts
const _serviceRegistry: Record<string, any> = {};

export function register(name?: string): ClassDecorator {
    return (target: any) => {
        const instance = new target();
        _serviceRegistry[name] = instance;  // 自动 new 并存入全局 Map
    };
}

// Service 是 Proxy，访问 Service.Editor 时从 _serviceRegistry 取值
export const Service = new Proxy({}, {
    get(_, prop) { return _serviceRegistry[prop]; },
});
```

```typescript
// src/core/scene/scene-process/service/node.ts
@register('Node')  // 类定义时立即执行，注册到 _serviceRegistry
export class NodeService extends BaseService {
    // 所有方法直接操作 cc.Node 引擎对象
}
```

子进程启动时注册到 RPC：
```typescript
// src/core/scene/scene-process/rpc.ts
async startup() {
    this.rpcInstance = new ProcessRPC<IMainModule>();
    this.rpcInstance.attach(process);
    const { Service } = await import('./service/core/decorator');
    this.rpcInstance.register(Service);  // 把整个 _serviceRegistry 注册给 RPC
}
```

### Step 3: 主进程发起 RPC 请求

```typescript
// src/api/scene/scene.ts
async createByType(params: ICreateByNodeTypeParams): Promise<INodeInfo | null> {
    const result = await Rpc.getInstance()
        .request('Node', 'createByType', [params]);   // 跨进程调用
    return result ? DumpConverter.toNode(result) : null;
}
```

ProcessRPC.request() 内部实现：
```typescript
// src/core/scene/process-rpc/process-rpc.ts
request(module, method, ...rest) {
    return new Promise((resolve) => {
        const id = ++this.msgId;
        const req = { id, type: 'request', module, method, args };
        this.callbacks.set(id, (res) => resolve(res.result));
        this.process.send?.(req);   // process.send() 发给子进程
    });
}
```

### Step 4: 子进程接收并执行（真正调用引擎 API）

```typescript
// src/core/scene/process-rpc/process-rpc.ts
private async onMessage(msg: RpcMessage) {
    if (msg.type === 'request') {
        const target = this.handlers[msg.module];  // → _serviceRegistry
        const result = await target[msg.method](...(msg.args || []));
        this.reply({ id: msg.id, type: 'response', result });
    }
}
```

`NodeService.createByType()` 实际操作引擎对象：
```typescript
// src/core/scene/scene-process/service/node.ts
async createByType(params: ICreateByNodeTypeParams): Promise<INode | null> {
    // 1. 获取当前场景根节点（引擎对象）
    const currentScene = Service.Editor.getRootNode();

    // 2. 创建引擎 Node 或从 Prefab 实例化
    let resultNode: Node;
    if (assetUuid) {
        const { node } = await createNodeByAsset({ uuid: assetUuid });
        resultNode = node;     // 真实的 cc.Node 实例
    } else {
        resultNode = new cc.Node();  // 新建引擎对象
    }

    // 3. 设置属性（引擎 API）
    resultNode.setParent(parent);
    resultNode.setPosition(params.position as Vec3);
    resultNode.name = params.name;

    // 4. 序列化返回给主进程
    return sceneUtils.generateNodeDump(resultNode) as INode;
}
```

### Step 5: 结果序列化返回

子进程通过 `process.send()` 把 `generateNodeDump()` 生成的 JSON 发回主进程，
主进程用 `DumpConverter.toNode()` 格式化为 `INodeInfo` 返回给 AI。

### 为什么用独立子进程

| 原因 | 说明 |
|------|------|
| 引擎隔离 | `require('cc')` 修改大量全局状态，独立进程避免污染 CLI 主进程 |
| 崩溃隔离 | 场景编辑崩溃不杀死 CLI 主进程和 MCP 服务器 |
| 热重启 | 可在不重启 CLI 的情况下 reload 子进程（切换场景时） |
| 渲染需要 | 子进程持有 GPU 上下文（OpenGL/WebGL headless），做离屏渲染 |
| MCP 可用 | MCP 在主进程正常响应，不受引擎状态影响 |

### 引用代理模式（Proxy Pattern）

主进程不持有 `cc.Node` 对象，只持有引用（uuid/path）。这是典型的跨进程引用代理：

- 子进程 = **执行者**：真正操作引擎对象（new、挂树、改属性）
- JSON 返回值 = **身份证**：记录 uuid/path 等元信息，不包含引擎对象本身
- 主进程 = **指挥官**：拿着上一步的身份证号，下发下一步命令

```typescript
// AI 开发流程示例：
createByType()        → 返回 { uuid: 'a1b2c3', path: '/Canvas/Player' }
addComponent({ uuid: 'a1b2c3', component: 'cc.BoxCollider2D' })  // 拿 uuid 继续操作
saveScene()           → 持久化到 assets/xxx.scene 文件
```

> **注意**：`createByType` 不写文件，只改子进程内存。需要显式调用 `scene-save` 才会把场景序列化写入磁盘。子进程崩溃或关闭时，未保存的修改会丢失。

### TypeScript 可选属性 `_task?: IBuilder`

`?` 是 TS 可选属性标记，表示属性可以是 `undefined`。等价于 `_task: IBuilder | undefined`。

在 `BundleManager` 中，`_task` 的两种场景：
- 完整构建流程中有 `BuildTask` 实例（`_task` 有值，使用构建系统路径）
- 独立测试/单独初始化时不传（`_task` 为 `undefined`，使用本地路径）

```typescript
config.scriptDest = this._task
    ? join(this._task.result.paths.bundleScripts, config.name, BuildGlobalInfo.SCRIPT_NAME)
    : join(config.dest, BuildGlobalInfo.SCRIPT_NAME);
```
