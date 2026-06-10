# 模块与关键实现

## 1. 主进程模块

### 1.1 `src/main/index.ts`

这是项目的总控入口，承担以下职责：

- 应用初始化与窗口创建
- Settings 读写与结构归一化
- IPC 注册
- Provider Hub 拉取与安装
- 引擎启动、停止、状态同步
- 框选向导调用与区域持久化
- Skill HTTP Server 启动

关键函数：

- `createWindow()`：创建主窗口
- `createSettingsWindow()`：创建设置窗口
- `startEngineCore()`：装配 Provider、Device、Session、Runtime 并启动引擎
- `stopEngineCore()`：停止运行时
- `buildDevice()`：根据 `appType` 和策略选择 `RPADevice` 或 `BoxSelectDevice`
- `normalizeSettings()`：兼容旧配置并产出统一 `AppSettings`
- `fetchProviderHub()`：拉取远端 Provider 目录并缓存

设计说明：

- 该文件是主进程的应用编排层，而不是底层算法层。
- 业务复杂度主要来源于配置归一化、扩展能力装配和 IPC 协调。

### 1.2 `src/main/provider-bundle.ts`

负责 Provider 的安装、校验、动态加载与实例化。

关键职责：

- 下载或读取 `manifest.json`
- 校验 manifest 结构
- 安装到用户目录
- 动态加载 bundle 文件
- 校验配置项
- 创建 `ProviderAdapter`

关键函数：

- `installProviderFromUrl()`：从 `http/https/file` 地址安装 Provider
- `getInstalledProviderManifest()`：读取已安装 Provider 的 manifest
- `loadInstalledProvider()`：加载并实例化已安装 Provider
- `loadBuiltinDoubaoProvider()`：加载内置默认 Provider
- `validateProviderConfig()`：校验用户填写的 Provider 配置
- `validateManifest()`：严格验证 Provider manifest 结构

设计说明：

- 这是项目插件化能力的核心。
- 主程序只要求 Provider 遵守统一事件协议，不要求固定模型厂商。

### 1.3 `src/main/skill-server.ts`

本地 HTTP 控制入口，供外部系统触发引擎启停。

暴露端点：

- `POST /skill/start`
- `POST /skill/pause`
- `GET /skill/status`

关键设计：

- 仅监听 `127.0.0.1`
- 有并发锁，避免重复 start/pause
- 与主进程已有 `startEngineCore()` / `stopEngineCore()` 复用同一套逻辑

### 1.4 `src/main/overlay-window.ts`

负责透明覆盖窗口与框选向导流程，包括：

- 创建 overlay 窗口
- 控制框选步骤顺序
- 回收 `contactList`、`chatMain`、`inputBox` 三个区域

### 1.5 `src/main/permission.ts`

负责 macOS 下权限检查和申请，例如：

- 屏幕录制
- 辅助功能

## 2. 核心运行时模块

### 2.1 `src/core/runtime-host.ts`

`RuntimeHost` 是运行时总线，控制事件队列、定时器和生命周期。

关键成员：

- `running`：当前是否运行
- `queue`：待处理事件队列
- `timers`：延迟任务集合
- `context`：包含 `appType`、状态对象和控制器

关键方法：

- `startSession()`：启动会话，触发 `channel.onStart`
- `stopSession()`：停止会话，清理 timer 和队列
- `enqueue()`：投递事件
- `schedule()`：延迟投递事件
- `drainQueue()`：串行消费事件

为什么重要：

- 这是所有业务动作最终的调度入口。
- 它保证状态机事件串行执行，减少桌面自动化中的竞争条件。

### 2.2 `src/core/generic-channel-session.ts`

`GenericChannelSession` 是主业务流程的状态机。

内部状态：

- `measuredAt`：最近一次布局测量时间
- `latestChatBaseline`：最近一次聊天区 baseline 建立时间
- `consecutiveUnreadFailures`：连续未读切换失败计数

核心事件：

- `bootstrap`
- `observe_chat`
- `provider.thinking`
- `provider.reply_text`
- `provider.skip`
- `provider.error`
- `check_unread`
- `wait_retry`

关键方法：

- `onStart()`：初始化设备、清理 baseline、投递 `bootstrap`
- `onEvent()`：驱动整个主循环
- `forwardProviderEvents()`：消费 Provider 输出并映射为会话事件
- `tryOpenUnreadConversation()`：尝试切换到有未读消息的会话

设计说明：

- 它是“统一业务逻辑”的核心，真正屏蔽了不同 IM 软件与不同布局测量策略的差异。

### 2.3 `src/core/session-types.ts`

该文件定义统一协议，是多个模块之间的契约层。

核心类型：

- `ProviderInput`
- `ProviderEvent`
- `SessionEvent`
- `ProviderAdapter`
- `RuntimeHostControls`
- `ChannelSession`

设计意义：

- 让主循环、设备层、Provider 层之间只通过协议通信
- 降低模块间的直接耦合

## 3. 设备层模块

### 3.1 `src/core/device.ts`

定义 `DesktopDevice` 接口，是项目的“宿主能力抽象”。

接口能力分组：

- 配置：`setAppType()`、`setApiKey()`
- 生命周期：`onSessionStart()`、`onSessionStop()`
- 感知：`measureLayout()`、`screenshot()`、`hasUnreadMessage()`、`hasChatAreaChanged()`
- 动作：`sendMessage()`、`activeUnreadByClick()`、`clickUnreadContact()`

### 3.2 `src/core/rpa-device.ts`

`RPADevice` 是基于视觉模型和桌面自动化的真实设备实现，适合微信与企业微信。

关键职责：

- 初始化 `AIClient`
- 执行并行布局测量
- 调用截图能力
- 调用未读红点检测
- 建立与比较聊天区 baseline
- 调用输入动作发送消息

关键方法：

- `measureLayout()`：并行调用 `detectUnreadArea()` 与 `detectWechatLayout()`
- `screenshot()`：截取聊天区并转为 base64
- `hasUnreadMessage()`：粗粒度未读入口检测
- `isChatContactUnread()`：细粒度联系人未读检测
- `setChatBaseline()` / `hasChatAreaChanged()`：聊天区像素 diff
- `sendMessage()`：发送文本到输入框

### 3.3 `src/core/box-select-device.ts`

`BoxSelectDevice` 是手动框选实现。

关键职责：

- 将框选结果写入统一 `LayoutCache`
- 对当前聊天区建立 baseline
- 使用像素 diff 判断是否有新消息
- 使用输入框坐标发送消息

实现特点：

- 不依赖视觉密钥
- 默认不主动进行多会话未读切换
- 更适合通用 IM 的单会话监控

关键方法：

- `measureLayout()`：把框选结果转换为缓存布局
- `screenshot()`：仅截取聊天区
- `hasUnreadMessage()`：固定返回 `false`
- `setChatBaseline()` / `hasChatAreaChanged()`：维护本地聊天区基线
- `sendMessage()`：根据输入框中心点发送回复

## 4. AI 与 RPA 基础能力

### 4.1 `src/core/ai-client.ts`

统一封装所有模型调用。

主要用途：

- 聊天回复生成
- VLM 视觉检测
- 接口可用性测试

关键方法：

- `getReply()`：根据截图生成聊天回复
- `detectVision()`：按 prompt 返回视觉识别结果
- `callText()`：纯文本调用
- `testConnection()`：测试接口连通性
- `callAPI()`：底层 `/chat/completions` HTTP 调用

### 4.2 `src/core/rpa/`

该目录是底层原子能力层，不直接承载业务流程。

重要文件：

- `vision-utils.ts`：VLM 布局识别与 `LayoutCache` 管理
- `screenshot-utils.ts`：截图与区域裁剪
- `input-utils.ts`：鼠标、键盘、输入框写入
- `has-unread.ts`：红点与联系人未读检测
- `image-compare.ts`：聊天区像素差异比较
- `window-utils.ts`：窗口查找与定位
- `types.ts`：`AppType`、区域类型、抓取策略等基础类型

## 5. 前端与桥接模块

### 5.1 `src/preload/index.ts`

向渲染层暴露：

- `electron.invoke()`
- `electron.on()`
- `electron.send()`
- `osInfo.platform`

设计目标：

- 保持渲染层与 Electron 原生 API 的隔离
- 让 UI 通过有限通道访问 IPC

### 5.2 `src/renderer/src/App.tsx`

这是当前 UI 的主要承载文件，包含：

- 主控制台
- 设置窗口页签
- Provider 配置页
- 运行日志视图
- 引擎启停操作

从架构角度看，它更像“控制面板”而不是核心业务引擎。

### 5.3 `src/renderer/overlay/`

承载框选向导的独立 React 应用，之所以拆成独立入口，是因为它运行在独立透明窗口中，需要与主界面打包隔离。

## 6. 配置与资源模块

### 6.1 `resources/providers/volcengine-ark/`

内置默认 Provider 示例。

包含：

- `manifest.json`
- `provider.bundle.js`

作用：

- 作为默认可用聊天 Provider
- 作为第三方接入实现参考

### 6.2 `docs/provider.md`

外部 Provider 的正式接入文档，约定：

- manifest 结构
- bundle 导出格式
- Provider 输入输出事件
- 安装方式

## 7. 最重要的类与函数清单

| 类/函数 | 所在模块 | 作用 |
| --- | --- | --- |
| `RuntimeHost` | `src/core/runtime-host.ts` | 运行时事件调度与生命周期控制 |
| `GenericChannelSession` | `src/core/generic-channel-session.ts` | 自动回复主状态机 |
| `DesktopDevice` | `src/core/device.ts` | 设备能力统一抽象 |
| `RPADevice` | `src/core/rpa-device.ts` | VLM + 自动化设备实现 |
| `BoxSelectDevice` | `src/core/box-select-device.ts` | 手动框选设备实现 |
| `AIClient` | `src/core/ai-client.ts` | 统一模型调用客户端 |
| `startEngineCore` | `src/main/index.ts` | 主进程引擎装配入口 |
| `buildDevice` | `src/main/index.ts` | 选择设备与策略 |
| `installProviderFromUrl` | `src/main/provider-bundle.ts` | 安装 Provider |
| `loadInstalledProvider` | `src/main/provider-bundle.ts` | 加载已安装 Provider |
| `startSkillServer` | `src/main/skill-server.ts` | 启动本地 Skill HTTP 服务 |
