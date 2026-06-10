# SightFlow Desktop Agent Code Wiki

## 1. 文档目标

这套 Wiki 用于帮助开发者快速理解 `sightflow-desktop-agent` 的代码结构、运行机制和扩展方式，覆盖以下内容：

- 项目整体架构与进程模型
- 主要模块职责与关键类/函数
- 运行时调用链与事件流
- 依赖关系与配置流转
- 本地开发、测试、构建与 Provider 扩展方式

## 2. 项目概览

SightFlow 是一个基于 Electron + React + TypeScript 的桌面端 AI RPA 客户端，目标是让桌面聊天软件具备“识别消息 -> 理解上下文 -> 自动生成回复 -> 写回输入框”的闭环能力。

项目支持两条布局测量路线：

- `VLM` 路线：面向微信、企业微信，依赖视觉模型识别桌面窗口结构。
- `Box Select` 路线：面向飞书、钉钉、Slack、Telegram 等，依赖用户手动框选联系人区、聊天区、输入框。

两条路线最终都汇总到统一的运行时抽象：

- `RuntimeHost`：运行时宿主与事件队列
- `GenericChannelSession`：会话状态机
- `DesktopDevice`：设备能力抽象
- `ProviderAdapter`：聊天回复生成器抽象

## 3. 技术栈

### 3.1 核心框架

- Electron 39
- React 19
- TypeScript 5
- electron-vite
- electron-builder

### 3.2 AI 与桌面自动化

- OpenAI 兼容视觉/对话接口：通过 `AIClient` 调用火山方舟
- `@hurdlegroup/robotjs`：鼠标/键盘自动化
- `active-win`、`get-windows`、`node-window-manager`：窗口识别
- `jimp`、`pngjs`、`pixelmatch`：截图处理与像素比对

## 4. 代码目录

```text
.
├─ build/                      # 打包资源
├─ docs/                       # 现有文档与本 Wiki
│  ├─ provider.md              # Provider 接入规范
│  └─ code-wiki/               # 本次生成的 Wiki
├─ resources/
│  └─ providers/               # 内置 Provider 资源
├─ scripts/                    # 开发辅助脚本与 CLI 测试入口
├─ src/
│  ├─ core/                    # 核心运行时、设备抽象、AI 客户端
│  │  └─ rpa/                  # 截图、视觉检测、输入控制、图像比较等底层能力
│  ├─ main/                    # Electron 主进程
│  ├─ preload/                 # 安全桥接层
│  └─ renderer/                # React 界面与框选向导
├─ electron.vite.config.ts     # 打包入口配置
├─ electron-builder.yml        # 应用打包配置
└─ package.json                # 脚本与依赖声明
```

## 5. 阅读顺序建议

建议按以下顺序阅读：

1. `01-architecture.md`：先看整体系统架构与数据流。
2. `02-modules.md`：再看主进程、核心运行时、设备层与前端模块职责。
3. `03-runtime-flows.md`：理解引擎启动、自动回复、未读检测、Provider 扩展链路。
4. `04-development.md`：最后查看启动方式、测试脚本、构建和开发注意事项。

## 6. 关键结论

- 项目的核心不是 UI，而是一个由 Electron 主进程编排的桌面自动化运行时。
- `DesktopDevice` 将“如何识别界面”和“如何执行输入动作”抽象出来，使 VLM 与框选模式能复用同一套会话逻辑。
- `Provider` 将“如何从截图生成回复”独立成可安装扩展，主程序只负责调度与执行。
- `Renderer` 主要承担配置、日志展示、框选操作入口和 Provider 管理，不承载核心业务状态机。
