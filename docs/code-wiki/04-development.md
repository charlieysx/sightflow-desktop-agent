# 开发、运行与扩展

## 1. 开发环境

建议环境：

- Node.js 18+
- npm
- 支持 Electron 桌面开发的操作系统环境
- 推荐 VS Code + ESLint + Prettier

项目基于 `electron-vite`，构建产物为：

- Main bundle
- Preload bundle
- Renderer bundle
- Overlay renderer bundle
- Test CLI bundle

## 2. 常用命令

### 2.1 安装依赖

```bash
npm install
```

### 2.2 本地开发

```bash
npm run dev
```

说明：

- 会通过 `scripts/dev-launch.mjs` 启动 `electron-vite dev`
- 启动后打开主界面
- 需要先选择目标应用，再完成必要配置与框选

### 2.3 类型检查

```bash
npm run typecheck
```

### 2.4 Lint 与格式化

```bash
npm run lint
npm run format
```

### 2.5 生产构建

```bash
npm run build
npm run build:win
npm run build:mac
npm run build:linux
```

## 3. 打包入口

`electron.vite.config.ts` 中定义了多入口打包：

### 3.1 Main 入口

- `src/main/index.ts`
- `scripts/test-cli.ts`

### 3.2 Renderer 入口

- `src/renderer/index.html`
- `src/renderer/overlay.html`

设计原因：

- 主窗口和框选向导窗口需要不同的 renderer bundle
- 测试 CLI 需要独立的 main bundle 入口

## 4. 运行前置条件

项目运行依赖以下条件：

### 4.1 视觉接口密钥

以下情况必须配置 `vision.apiKey`：

- 抓取策略为 `vlm`
- 当前使用内置 Doubao Provider
- 当前安装的 Provider 仍共享视觉密钥

### 4.2 目标应用窗口

如果走 `RPADevice`：

- 微信或企业微信必须已启动
- 窗口不能被完全遮挡或最小化
- 窗口布局需要足够稳定，便于模型识别

### 4.3 框选区域

如果走 `BoxSelectDevice`：

- 必须已保存 `contactList`
- 必须已保存 `chatMain`
- 必须已保存 `inputBox`

## 5. 测试能力

项目内置了一些偏集成性质的测试入口，更接近“调试脚本”而非纯单元测试。

### 5.1 命令行测试脚本

```bash
npm run dev:test-screenshot
npm run dev:test-reply
npm run dev:test-switch
```

它们会先执行构建，再以 `TEST_MODE` 方式启动 `out/main/test-cli.js`。

### 5.2 RPA 测试目录

`src/core/rpa/tests/` 下包含：

- `test-screenshot.ts`
- `test-reply.ts`
- `test-switch.ts`
- `test-vlm-parallel.ts`

这类测试更适合验证：

- 截图是否成功
- 模型调用是否可用
- 会话切换是否正确
- 并行 VLM 检测是否比串行更稳定或更快

## 6. Provider 扩展方式

项目将聊天回复能力设计成可安装 Provider。

### 6.1 Provider 包结构

```text
provider-root/
  manifest.json
  provider.bundle.js
```

### 6.2 manifest 关键字段

- `apiVersion`
- `id`
- `name`
- `version`
- `entry`
- `moduleType`
- `capabilities`
- `configSchema`

### 6.3 支持的配置字段类型

- `string`
- `password`
- `select`
- `boolean`

### 6.4 安装来源

支持：

- `https://.../manifest.json`
- `file:///absolute/path/to/manifest.json`

### 6.5 加载策略

- `moduleType=module` 时使用 ESM `import`
- `moduleType=commonjs` 时使用 `require`
- 未声明时按老规则根据扩展名判断

## 7. 数据持久化

项目主要持久化两类数据：

### 7.1 Settings

由 `electron-store` 存储，包括：

- 语言
- 当前目标应用
- 视觉密钥
- Provider 安装信息
- Provider 配置
- 抓取策略
- 每个应用的框选区域

### 7.2 Provider 安装包

安装目录位于用户数据目录：

```text
<userData>/providers/<provider-id>/<version>/
```

其中会保存：

- `manifest.json`
- Provider bundle 入口文件

## 8. 开发时需要重点关注的文件

如果要理解项目核心行为，优先看这些文件：

1. `src/main/index.ts`
2. `src/core/runtime-host.ts`
3. `src/core/generic-channel-session.ts`
4. `src/core/device.ts`
5. `src/core/rpa-device.ts`
6. `src/core/box-select-device.ts`
7. `src/main/provider-bundle.ts`
8. `src/core/ai-client.ts`

如果要修改 UI，再看：

1. `src/renderer/src/App.tsx`
2. `src/preload/index.ts`
3. `src/renderer/overlay/`

## 9. 已知实现特点与维护建议

### 9.1 实现特点

- 主进程业务较重，承担了编排、配置、插件加载和部分协议转换
- `App.tsx` 体量较大，后续可以考虑拆分为控制台、设置页、Provider 页等子组件
- `BoxSelectDevice` 当前更偏单会话模式，不是完整多会话自动巡检器

### 9.2 维护建议

- 优先通过 `DesktopDevice` 和 `ProviderAdapter` 扩展能力，不要把新逻辑直接塞进 `GenericChannelSession`
- 若新增聊天应用，先判断是适合 `vlm` 还是 `box-select`
- 若新增模型供应商，优先做成独立 Provider，而不是修改主程序内置逻辑
- 若要增强可靠性，优先补足截图、窗口定位和 diff 检测链路的诊断日志

## 10. 新成员上手建议

建议用以下顺序熟悉仓库：

1. 运行 `npm install` 和 `npm run dev`
2. 阅读 `docs/code-wiki/README.md`
3. 阅读 `01-architecture.md` 和 `03-runtime-flows.md`
4. 结合 `src/main/index.ts` 与 `src/core/generic-channel-session.ts` 对照代码
5. 手动完成一次框选并跑通启动链路
6. 再尝试安装或开发一个自定义 Provider
