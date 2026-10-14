# lolPing 二次开发指南:AI 识图功能案例

> 基于 darrenprx/lolPing v0.5.x(MIT)。本文以「连按问号 AI 截图识别」功能的完整落地过程为例,讲清这个仓库的架构、约定与二开方法论。

## 一、项目是什么

桌面版 LOL 信号轮盘:按住 Alt 拖动,在屏幕任意位置发出原版信号动画+音效;表情轮盘、好友房间联机(局域网 + Nostr 公共中继)建立在同一套轮盘之上。支持 Windows 11 x64 与 macOS 12+(Apple Silicon)。

**技术栈**:Electron 44 + TypeScript(strict)+ electron-vite + C++ 原生输入助手 + React 19(Fluent UI v9 设置页)+ Vitest。

## 二、整体架构(四层)

```
┌─────────────────────────────────────────────────────────────┐
│ native/hook-helper(C++,Win 低级钩子 / Mac 事件点击)           │
│  decision.cpp 手势状态机:触发键+拖拽 → wheelOpen/Move/Release  │
│  与 Electron 以 stdin/stdout JSON 行通信(protocol.ts)         │
├─────────────────────────────────────────────────────────────┤
│ src/main(主进程)                                            │
│  InputBridge ←→ helper;OverlayManager(每显示器一个透明覆盖窗) │
│  SettingsStore(本地 JSON);RoomManager(Nostr+LAN 双传输)      │
│  Tray / Updater / CaptureManager / AnswerManager /            │
│  QuestionPingDetector / aiVision   (后四者为 AI 识图新增)      │
├─────────────────────────────────────────────────────────────┤
│ src/preload(每个窗口一座最小桥梁,沙箱化)                      │
│  overlay / settings / capture / answer                        │
├─────────────────────────────────────────────────────────────┤
│ src/renderer                                                  │
│  overlay/(原生 TS+DOM:轮盘、信号动画、toast)                  │
│  settings/(React 设置页)                                      │
│  capture/(微信式选区页)/ answer/(答案展示页)                  │
├─────────────────────────────────────────────────────────────┤
│ src/shared(纯代码,任何进程可用,禁止 Electron/Node 依赖)      │
│  settings(含 normalizeSettings 归一化)/ ipc(通道与类型)/      │
│  i18n(中英双语)/ pings / protocol / roomProtocol              │
└─────────────────────────────────────────────────────────────┘
tests/(Vitest,镜像 src/ 结构)· site/(浏览器 demo)· tools/(资产提取、联机测试)
```

### 铁律(AGENTS.md 保保证,改代码前必读)

1. **输入直通**:非手势的按键/点击必须原样到达下层应用,不得冻结、丢失、乱序。
2. **双平台**:平台差异用 `shared/platform.ts` 分支,或放进 `hooks_win.cpp`/`hooks_mac.cpp`。
3. **双语**:所有用户可见文案进 `shared/i18n.ts`,中英各一条;README 改动两份同步。
4. **向后兼容**:新设置必须在 `normalizeSettings` 有默认值;房间协议老版本可读。
5. **网络可见**:新增网络主机必须在 README 隐私说明中写明(AI 功能新增了用户自配的 API 地址,默认 `api.deepseek.com`)。
6. **逻辑进纯模块**:决策逻辑放在可注入时间(`now: () => number`)的纯类里,Electron/DOM 胶水保持薄。

## 三、AI 识图数据流

```
'shared' 事件(用户发的每个信号,预览和远程信号不算)
  → QuestionPingDetector:滑动窗口计数(5s 内 N 次,阈值/窗口/目标信号可配)
  → 达阈值触发(10s 冷却防连触) → startAiCapture()
  → 未配 Key?角落 toast 提示。配了 → CaptureManager.start(信号所在显示器)
      → helper 挂起(suspend),每显示器一个置顶选区窗,焦点给触发屏
      → 微信交互:单击=全屏 / 拖拽=框选+✓✕工具条 / Esc 取消 / 60s 无人操作自动关
  → confirm(displayId, rect) → desktopCapturer 截屏 → NativeImage.crop → JPEG(质量 90)
  → aiVision.recognizeImage():OpenAI 兼容 /chat/completions,
      图片 base64 data URL + 中性提示词"请识别并解释这张截图的内容。"
  → AnswerManager:全屏遮罩(点击/Esc 关)或角落小窗(悬停保留,10s 自动关,可配)
      加载态/错误态(未配 Key、限流、模型拒绝、超时、网络失败各有专属文案)
```

## 四、新增/修改文件清单(AI 识图)

| 类别 | 文件 | 内容 |
|---|---|---|
| 共享 | `src/shared/settings.ts` | 10 个 `ai*` 设置字段 + LIMITS + 归一化(baseUrl 校验 http(s)、模型名字符集、Key 截断) |
| 共享 | `src/shared/i18n.ts` | 中英各 ~45 条新文案 |
| 共享 | `src/shared/ipc.ts` | Capture/Answer 通道与类型、`AiTestResult`、`SettingsApi.testAi` |
| 主进程 | `src/main/questionPingDetector.ts` | 纯逻辑:滑动窗口计数器(时间注入) |
| 主进程 | `src/main/aiVision.ts` | 纯逻辑:OpenAI 兼容视觉请求(fetch 注入)、错误分类、`aiErrorText`、1px 测试图 |
| 主进程 | `src/main/captureManager.ts` | 选区窗口编排、desktopCapturer 截屏裁剪、helper 挂起/恢复、60s 超时 |
| 主进程 | `src/main/answerManager.ts` | 答案窗口(全屏/小窗)、状态追赶(见踩坑 2) |
| 主进程 | `src/main/index.ts`、`settingsIpc.ts`、`paths.ts` | 接线、`ai:test` 处理器、Page 类型 |
| 预加载 | `src/preload/capture.ts`、`answer.ts` | 最小桥梁;**通道名内联**(见踩坑 1) |
| 渲染 | `src/renderer/capture/`、`answer/` | 选区页与答案页(html/ts/css) |
| 渲染 | `src/renderer/settings/components/AiSection.tsx`、`App.tsx` | 设置分区与导航 |
| 测试 | `tests/main/questionPingDetector.test.ts`、`aiVision.test.ts`、`tests/shared/settings.test.ts` | 27 个新用例 |

## 五、踩坑记录(二开此项目必看)

1. **沙箱 preload 禁止 require 兄弟文件。** 新 preload 与 overlay 共享了 `shared/ipc.ts` 的运行时常量,Rollup 拆出 `chunks/ipc-*.js`,沙箱加载失败 → **整个轮盘/信号系统瘫痪**(overlay 页面 `window.overlay` 缺失)。`settings.ts` 的注释里早有警告("Inlined rather than imported")。**规矩:preload 里只允许类型导入 + 就地内联常量。** 验证手段:构建后确认 `out/preload/` 下无 `chunks/` 目录。
2. **置顶窗口的消息要在 `did-finish-load` 之后发。** `capture:begin`/`answer:state` 在 `loadPage()` 后立即发送,页面没加载完就丢;窗口显示了但内容空白(root 保持 hidden)。修法:manager 缓存最近状态,`did-finish-load` 时补发。
3. **渲染层初始 `hidden` 别忘了摘。** answer 页 `<div id="root" hidden>`,状态处理器只改了 className 没去掉 hidden——数据都到了(DevTools 里能看到 AI 答案文本)屏幕上却永远空白。
4. **中文 Windows(代码页 936)编译 C++ 会报 C4819**。源码是 UTF-8,MSVC 默认按系统代码页读。`build.cmd` 加 `/utf-8`。
5. **MSVC 传统预处理器对"宏参数里的 raw string literal"错误分词**(`helper_tests.cpp` 的 `CHECK(parseFlatJson(R"(...)"))` 报非法转义)。测试编译行加 `/Zc:preprocessor`(一致性预处理器)。
6. **npm install 后 electron 二进制可能没下载**(国内网络):`ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/ node node_modules/electron/install.js`。
7. **调试利器**:`npm run dev -- -- --remote-debugging-port=9222` 后,CDP `/json` 能看到每个窗口(含 overlay/capture/answer),可直接查询页面内部状态、注入监听器——比截图猜快得多。

## 六、验证结论

- `npm run typecheck` ✓;`npm test` 全量通过(含本次新增 27 个用例);`npm run test:helper` 444/444;`npm run build` ✓。
- 真机端到端(合成输入驱动真实钩子链路):连按 3 次危险信号("?")→ 微信式遮罩出现 → 单击全屏截图 → 截图 275 KB → DeepSeek `deepseek-flash` 返回 1328 字符的中文识别 → 全屏遮罩卡片展示 → 点击/Esc 关闭。
- 已知限制:独占全屏游戏会盖住选区/答案窗口(与信号层同一限制),游戏内使用请选无边框/窗口化模式。

## 七、日常命令

```bash
npm run dev            # 开发(可选 -- --remote-debugging-port=9222)
npm run typecheck      # TS 严格检查
npm test               # Vitest 全量
npm run build:helper   # C++ 输入助手(需 MSVC)
npm run test:helper    # C++ 测试
npm run build          # 产出 out/
npm run dist           # 打 Windows 安装包(electron-builder)
```

## 八、后续二开切入点建议

1. **触发体验**:wheel 释放第 N 次时抑制最后一次动画;触发后遮罩直接进入"上一次框选的区域"快捷模式。
2. **答案增强**:小窗支持 markdown 渲染;"追问"输入框(桌面场景打字可用,游戏内仍建议只识图);答案历史(本地存最近 N 条截图+答案,设置页可回看)。
3. **多轮对话**:aiVision 目前单轮;把历史消息串起来即可支持追问。
4. **预设固定区域**:先框选一块区域存起来,触发时跳过选区直接截那块(适合装备栏/地图)。
