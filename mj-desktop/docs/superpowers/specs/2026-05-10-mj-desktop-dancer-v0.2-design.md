# MJ Desktop Dancer v0.2 设计文档

日期：2026-05-10  
范围：在现有 MVP（透明置顶窗口 + GIF 展示 + 拖动 + 麦克风音频节拍触发）基础上实现 PRD v0.2（托盘、形象切换、BPM、配置持久化、设置面板）。  
平台重点：macOS Apple Silicon（ARM64）优先，保持跨平台可运行（Windows/macOS/Linux）。

## 目标与非目标

### 目标（v0.2）

- 配置持久化：窗口位置、当前形象、节拍灵敏度/参数（CONFIG 全量字段）。
- 系统托盘图标：左键切换显示/隐藏；右键菜单（退出/穿透/重置位置/设置/形象切换）。
- 设置面板：独立窗口，可视化调节节拍检测 CONFIG，并即时生效。
- 多形象切换：扫描 `assets/avatars/*.gif`，菜单与设置面板可切换。
- BPM 估算：基于节拍间隔的中位数滤波估算 BPM，并显示在主窗口状态栏/设置面板。

### 非目标（v0.2 不做）

- 系统音频直采（WASAPI loopback / BlackHole 引导）仍由用户按 README 自行配置录音设备。
- 3D 模型（Three.js/MMD）与打包（electron-builder）属于后续版本。
- 复杂节拍分级动作库：仅保留现有缩放+亮度反馈，后续再扩展。

## 技术约束与原则

- Electron ≥ 28（项目当前 Electron 30）。
- 渲染侧：原生 HTML/CSS/JS，不引入 React/Vue。
- 音频：Web Audio API，不引入 Tone.js/librosa。
- 代码组织：主进程模块放 `src/main/`，渲染进程放 `src/renderer/`；节拍算法集中到 `src/renderer/beatDetector.js` 便于单测。
- 不在日志中输出敏感信息（无额外 secret 输入）。

## 目录结构（v0.2 目标形态）

```
mj-desktop/
├── package.json
├── main.js
├── preload.js
├── index.html
├── styles.css
├── settings.html
├── assets/
│   ├── placeholder.svg
│   ├── trayTemplate.png (可选：如提供真实图标资源)
│   └── avatars/
│       ├── xxx.gif
│       └── yyy.gif
└── src/
    ├── main/
    │   ├── app.js
    │   ├── store.js
    │   └── avatars.js
    └── renderer/
        ├── renderer.js
        ├── beatDetector.js
        └── bpmEstimator.js
```

说明：
- `main.js`/`preload.js` 保持入口职责，业务逻辑迁移到 `src/`。
- `settings.html` 是独立渲染页；对应 JS/CSS 可内联或拆文件（保持轻量）。

## 核心数据模型（electron-store）

键空间（建议）：
- `windowBounds`: `{ x, y, width, height }`
- `mouseThrough`: `boolean`
- `selectedAvatar`: `string`（例如 `xxx.gif`，仅文件名）
- `beatConfig`:  
  - `fftSize: number`
  - `bassRange: [number, number]`
  - `historySize: number`
  - `beatThreshold: number`
  - `minEnergy: number`
  - `beatCooldownMs: number`

默认值来源：
- `beatConfig` 默认取现有 MVP `CONFIG`。
- `selectedAvatar` 默认取 `assets/avatars/` 扫描结果第一项；若目录为空，则回退到 `assets/mj-dance.gif`，再回退到 `assets/placeholder.svg`。

一致性策略：
- 写入 store 前做最小校验（数值范围、bassRange 顺序、fftSize 合法值）。
- 主窗口与设置窗口以主进程为“真源”，通过 IPC 下发当前配置、广播变更。

## 主进程设计

### 窗口

1) 主窗口（MJ）
- 透明、无边框、置顶、不可调整大小、跳过任务栏。
- 记录与恢复 `windowBounds`。
- 支持“鼠标穿透”切换：`setIgnoreMouseEvents(mouseThrough, { forward: true })`。

2) 设置窗口（Settings）
- 普通窗口（非透明），可关闭隐藏；避免影响主窗口置顶体验。
- 仅一份实例：若已存在则聚焦。

### 托盘

- Tray 实例常驻。
- 左键点击：切换主窗口 show/hide（隐藏时不退出进程）。
- 右键菜单项（建议顺序）：
  - 显示/隐藏
  - 设置…
  - 形象（子菜单，来自 `assets/avatars/*.gif` 扫描）
  - 鼠标穿透（勾选态）
  - 重置位置
  - 退出

图标策略（macOS 兼容）：
- 优先使用 `nativeImage.createFromPath(assets 内 png)`；若资源缺失，使用 `nativeImage.createFromDataURL` 创建一张极简模板图（黑白）作为 fallback。
- macOS 上可设置模板图属性（若资源满足要求）。

### IPC 通道（建议）

- `config:get` → 返回全量配置（beatConfig、selectedAvatar、mouseThrough、windowBounds）
- `config:updateBeatConfig` → 主进程校验后写 store，并广播 `config:beatConfigChanged`
- `config:setAvatar` → 写 store，并广播 `config:avatarChanged`
- `window:resetPosition`
- `window:toggleMouseThrough`
- `settings:open`
- `app:quit`

广播策略：
- 主进程对主窗口、设置窗口都广播变更，确保两端 UI 同步。

## 渲染进程设计

### 主窗口（index.html）

- 资源加载：根据主进程下发 `selectedAvatar` 决定 `<img src>`：
  - `assets/avatars/<selectedAvatar>`（存在时）
  - fallback：`assets/mj-dance.gif`（若保留）
  - fallback：`assets/placeholder.svg`
- 节拍检测：把音频采集与算法编排放在 `renderer.js`，把“能量→beat 判定”放在 `beatDetector.js`。
- BPM：在每次 beat 事件触发时更新 `bpmEstimator`，将 BPM 显示在 `#status`（例如 `BEAT (energy) | BPM 123`）。

### 设置窗口（settings.html）

- 启动时通过 `config:get` 拉取当前配置并填表。
- 任一项改变时立即触发 `config:updateBeatConfig`（可做 150-300ms debounce）。
- 显示当前 avatar，并提供下拉选择（与托盘菜单同源列表）。
- “恢复默认”按钮：将 beatConfig 重置为默认值（由主进程提供默认常量）。

### beatDetector.js（边界）

输入：
- `analyser`（Web Audio AnalyserNode）
- `config`（beatConfig）
- 回调：`onBeat({ energy, ts })`、`onFrame({ bass, avg, ts })`（可选用于 UI debug）

输出：
- 只输出 beat 事件与可选 frame 数据，不直接操作 DOM。

### BPM 估算（bpmEstimator.js）

算法（轻量、实时、抗抖）：
- 在每次 beat 时记录间隔 `dt`（ms）。
- 维护固定长度窗口（例如最近 12-24 个间隔）。
- 对间隔做中位数滤波得到 `dtMedian`，再 `bpm = 60000 / dtMedian`。
- BPM 合理范围裁剪（例如 40-220），超出则不更新显示或降权。

## 跨平台与 macOS ARM 适配说明

- Electron 30 对 macOS arm64 原生支持；依赖不引入 native module（electron-store 为纯 JS），避免架构编译问题。
- 透明窗口在不同 WM 表现差异属于已知限制；macOS 上通常稳定。
- 录系统声音仍依赖用户设备路由（BlackHole 等），v0.2 不做额外引导安装，但可在设置面板/README 提示。

## 验收标准（v0.2）

- `npm install && npm start` 可启动，无控制台报错。
- 主窗口：透明置顶、可拖动、节拍触发缩放+亮度反馈正常。
- 托盘：左键显示/隐藏；右键菜单功能齐全（退出/穿透/重置位置/设置/形象切换）。
- 配置持久化：重启后窗口位置、形象选择、节拍参数保持一致。
- 设置面板：修改参数后即时生效（能明显改变触发灵敏度），并持久化。
- 多形象：在 `assets/avatars/` 放入多个 GIF，可在菜单中切换并立即生效。
- BPM：状态栏持续显示可更新的 BPM（有节拍输入时）。
