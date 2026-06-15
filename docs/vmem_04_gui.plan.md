---
name: "VMem 子方案 ④ - WPF GUI"
overview: "基于 WPF UI (Fluent Design) 构建的现代化图形界面。采用 MVVM 架构，包含卡片式仪表盘、场景化创建向导、实时性能监控及系统托盘集成。"
todos:
  - id: gui-framework
    content: "引入 WPF UI (lepoco) 框架，搭建 Windows 11 Fluent Design 基础外壳（Mica 材质、暗黑模式）"
    status: pending
  - id: gui-navigation
    content: "实现 NavigationView 导航框架（仪表盘、性能监控、设置）"
    status: pending
  - id: gui-dashboard
    content: "实现 Dashboard 视图：卡片式磁盘列表（环形进度条、快捷操作按钮）"
    status: pending
  - id: gui-create-wizard
    content: "实现创建磁盘向导：支持场景预设（浏览器缓存/系统TEMP/自定义）与高级选项折叠"
    status: pending
  - id: gui-monitor
    content: "实现 Performance 视图：集成 LiveCharts2 绘制实时吞吐量与 IOPS 折线图"
    status: pending
  - id: gui-tray
    content: "实现系统托盘与快速操作菜单（Hardcodet.NotifyIcon.Wpf）"
    status: pending
  - id: gui-ipc-client
    content: "实现 IpcClientService，与后台 Service 进行异步通信及事件订阅"
    status: pending
isProject: false
---

# 子方案 ④ - WPF GUI 深度设计

> 父方案：[VMem RAM Disk Design](./vmem_ram_disk_design_031231c8.plan.md)
> 对应代码：`src/VMem.App/`
> Phase：2（基础界面与交互）、3（性能监控与 IPC 联调）

为了提供符合现代审美的用户体验（UX），摒弃传统的 DataGrid 和原生 WPF 控件，本项目将采用 **Windows 11 Fluent Design** 设计语言，并从 PowerToys、Rufus 以及 galpt/temp 等优秀开源项目中汲取灵感。

---

## 0. 图标资源（SVG 设计）

所有图标均为 SVG 矢量格式，存放于 `assets/icons/`。详见 [图标索引](../assets/icons/README.md)。

| 类别 | 图标文件 | 用途 |
|------|---------|------|
| 应用 | `vmem-app.svg` | 主窗口标题栏、任务栏、开始菜单 |
| 托盘 | `vmem-tray.svg` / `-connected` / `-disconnected` | 系统托盘状态切换 |
| 导航 | `nav-dashboard.svg` / `nav-performance.svg` / `nav-settings.svg` / `nav-about.svg` | NavigationView 左侧菜单 |
| 操作 | `action-create.svg` / `action-open-folder.svg` / `action-eject.svg` / `action-delete.svg` | 磁盘卡片按钮、工具栏 |
| 状态 | `status-connected.svg` / `status-disconnected.svg` / `status-warning.svg` | 底部状态栏、卡片徽章 |
| 预设 | `preset-browser.svg` / `preset-temp.svg` / `preset-dev.svg` | 创建向导场景预设图标 |
| 功能 | `disk-drive.svg` / `disk-mounted.svg` / `memory-chip.svg` / `speed-gauge.svg` / `snapshot.svg` | 卡片图标、监控面板 |
| 工具 | `tool-log.svg` / `tool-repair.svg` / `tool-export.svg` | 设置页诊断区域 |
| 主题 | `theme-auto.svg` | 设置页主题切换 |
| 服务 | `service.svg` | 设置页后台服务区域 |
| 安装 | `installer.svg` | Inno Setup 安装向导 |

**WPF 集成方式**：

```xml
<!-- XAML 中引用 SVG（通过 SharpVectors 或编译时转 DrawingImage）-->
<ui:SymbolIcon Symbol="Home24" />  <!-- WPF UI 内置图标（导航等通用图标） -->

<!-- 自定义 SVG 图标通过资源字典加载 -->
<Image Source="/Assets/Icons/vmem-app.svg" Width="32" Height="32" />

<!-- 系统托盘图标动态切换 -->
<!-- Connected → vmem-tray-connected.ico -->
<!-- Disconnected → vmem-tray-disconnected.ico -->
<!-- ICO 由 SVG 在构建时通过 Inkscape+ImageMagick 转换生成 -->
```

---

## 1. UI 框架与架构选型

### 1.1 核心 UI 库：WPF UI (by lepoco)
- **为什么选择它**：目前 WPF 生态中实现 Windows 11 Fluent Design 最完美的开源库。
- **特性**：原生支持 Mica / Acrylic 背景材质、自动适配系统暗黑/明亮模式、提供现代化的 `NavigationView`、`Card`、`ContentDialog` 和 `Snackbar`。

### 1.2 架构：MVVM + 依赖注入
- **MVVM 框架**：`CommunityToolkit.Mvvm`（微软官方，支持 Source Generator，零样板代码）。
- **依赖注入**：`Microsoft.Extensions.Hosting`，与后台 Service 保持一致的 DI 体验。

```text
VMem.App/
├─ App.xaml / AppHost.cs         # 启动与 DI 容器配置
├─ Services/
│  ├─ IpcClientService.cs        # 封装 Named Pipe 通信，单例
│  └─ ThemeService.cs            # 主题切换服务
├─ ViewModels/
│  ├─ MainWindowViewModel.cs
│  ├─ DashboardViewModel.cs      # 仪表盘（磁盘列表）
│  ├─ PerformanceViewModel.cs    # 性能监控
│  └─ SettingsViewModel.cs       # 全局设置
└─ Views/
   ├─ MainWindow.xaml            # 包含 NavigationView 的主壳
   ├─ Pages/                     # 导航页面
   │  ├─ DashboardPage.xaml
   │  ├─ PerformancePage.xaml
   │  └─ SettingsPage.xaml
   └─ Components/                # 可复用组件（如 DiskCardControl）
```

---

## 2. 界面布局与交互设计 (UX)

以下是用字符画（ASCII Art）绘制的 VMem GUI 界面概念图，直观展示基于 Windows 11 Fluent Design（WPF UI 框架）的布局结构和交互逻辑。

### 2.1 主窗口：仪表盘 (Dashboard)
采用左侧 `NavigationView` 导航，右侧为卡片式（Card）磁盘列表。告别枯燥的表格，让磁盘状态一目了然。

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ ≡  VMem RAM Disk                                                 _ □ X   │
├────────┬─────────────────────────────────────────────────────────────────┤
│ 🏠 仪表盘 │  Dashboard                                              [+ 创建] │
│ 📊 性能   │                                                                 │
│ ⚙️ 设置   │  ┌──────────────────────────┐  ┌──────────────────────────┐     │
│          │  │ [ R: ] Browser Cache     │  │ [ T: ] System TEMP       │     │
│          │  │        VMEM              │  │        VMEM              │     │
│          │  │   ╭───╮                  │  │   ╭───╮                  │     │
│          │  │   │60%│  1.2 GB / 2.0 GB │  │   │15%│  0.6 GB / 4.0 GB │     │
│          │  │   ╰───╯                  │  │   ╰───╯                  │     │
│          │  │                          │  │                          │     │
│          │  │  [📂打开] [⚙设置] [⏹卸载]  │  │  [📂打开] [⚙设置] [⏹卸载]  │     │
│          │  └──────────────────────────┘  └──────────────────────────┘     │
│          │                                                                 │
│          │                                                                 │
│          │                                                                 │
│ ℹ️ 关于   │                                                                 │
├────────┴─────────────────────────────────────────────────────────────────┤
│ 🟢 Service: Connected  |  System RAM: [██████████░░░░░] 64% (20G/32G)    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 创建磁盘向导 (Create Dialog)
采用 `ContentDialog` 覆盖在主界面上方。提供“场景预设”降低小白认知门槛，同时保留“自定义”高级选项供极客使用。

**模式 A：一键预设（默认推荐）**
```text
      ┌────────────────────────────────────────────────────────┐
      │ 创建 RAM 盘                                          X │
      ├────────────────────────────────────────────────────────┤
      │ 模式选择：                                             │
      │ ◉ 一键预设 (推荐)          ○ 自定义                    │
      │                                                        │
      │ ┌────────────────────────────────────────────────────┐ │
      │ │ 🚀 浏览器缓存加速 (1GB, 4KB页)                     │ │
      │ │    自动映射 Chrome/Edge 缓存目录，提升网页加载速度 │ │
      │ ├────────────────────────────────────────────────────┤ │
      │ │ 🚀 系统 TEMP 盘 (2GB, 4KB页)                       │ │
      │ │    接管系统临时文件，减少固态硬盘磨损              │ │
      │ ├────────────────────────────────────────────────────┤ │
      │ │ 🚀 开发编译缓存 (4GB, 64KB页)                      │ │
      │ │    适合大文件吞吐，极大提升编译速度                │ │
      │ └────────────────────────────────────────────────────┘ │
      │                                                        │
      │ 盘符: [ R: ▼ ]                                         │
      │                                                        │
      │ ▶ 高级选项 (已折叠)                                    │
      │                                                        │
      │                                    [ 取消 ] [ ⭐ 创建 ] │
      └────────────────────────────────────────────────────────┘
```

**模式 B：自定义（展开高级选项）**
```text
      ┌────────────────────────────────────────────────────────┐
      │ 创建 RAM 盘                                          X │
      ├────────────────────────────────────────────────────────┤
      │ 模式选择：                                             │
      │ ○ 一键预设 (推荐)          ◉ 自定义                    │
      │                                                        │
      │ 容量: ├───■────────────────────────┤ [ 2048 ] [MB ▼]   │
      │ 卷标: [ VMem RAM Disk                            ]     │
      │ 盘符: [ R: ▼ ]                                         │
      │                                                        │
      │ ▼ 高级选项                                             │
      │   页大小:   [ 4 KB  ▼ ]                                │
      │   内核缓存: [☑] 启用 (极大提升读取性能)                │
      │   自动挂载: [☑] 开机自动创建此 RAM 盘                  │
      │   Root SDDL: [ Everyone:FA                       ]     │
      │                                                        │
      │                                    [ 取消 ] [ ⭐ 创建 ] │
      └────────────────────────────────────────────────────────┘
```

### 2.3 磁盘设置对话框 (Disk Settings Dialog)

点击磁盘卡片上的 [设置] 按钮弹出 `ContentDialog`，展示当前磁盘配置并允许修改：

```text
      ┌────────────────────────────────────────────────────────┐
      │ 磁盘设置 - R: Browser Cache                          X │
      ├────────────────────────────────────────────────────────┤
      │                                                        │
      │ 基本信息（只读，修改需卸载后重新创建）                │
      │   盘符:       R:                                       │
      │   容量:       2.0 GB                                   │
      │   页大小:     4 KB                                     │
      │   内核缓存:   已启用                                   │
      │   状态:       🟢 已挂载                                 │
      │                                                        │
      │ ─────────────────────────────────────────────────────  │
      │                                                        │
      │ 可修改设置                                             │
      │   卷标:       [ Browser Cache                    ]     │
      │   自动挂载:   [☑] 开机自动创建此 RAM 盘               │
      │                                                        │
      │ ─────────────────────────────────────────────────────  │
      │                                                        │
      │ 持久化                                                 │
      │   启用快照:   [☑] 定期保存数据到磁盘                  │
      │   快照周期:   [ 5 ▼] 分钟                              │
      │   镜像路径:   C:\ProgramData\VMem\snapshots\R.vmem     │
      │               [浏览...]                                │
      │   上次快照:   2026-06-16 00:05:32 (成功, 12.3MB)       │
      │                                                        │
      │ ─────────────────────────────────────────────────────  │
      │                                                        │
      │ 危险操作                                               │
      │   [🗑️ 卸载磁盘]  [ ] 同时从自动挂载移除                │
      │                                                        │
      │                                    [ 取消 ] [ 💾 保存 ] │
      └────────────────────────────────────────────────────────┘
```

**交互说明**：
- 点击 [保存] → IPC `UpdateDiskConfigRequest` 发送到 Service → Service 热修改生效 + 更新 config.json
- 点击 [卸载磁盘] → 弹出二次确认对话框 → IPC `RemoveDiskRequest`
- 基本信息区域为只读灰色文字，提示"修改需卸载后重新创建"
- 快照"镜像路径"默认自动生成 `%ProgramData%\VMem\snapshots\{DriveLetter}.vmem`，用户可自定义

### 2.4 卸载确认对话框

```text
      ┌────────────────────────────────────────────────────────┐
      │ ⚠ 确认卸载                                            │
      ├────────────────────────────────────────────────────────┤
      │                                                        │
      │  确定卸载 R: (Browser Cache) 吗？                      │
      │                                                        │
      │  当前状态：已用 1.2 GB / 2.0 GB，128 个文件            │
      │                                                        │
      │  ⚠ 卸载后磁盘上的所有数据将丢失！                     │
      │                                                        │
      │  [ ] 同时从自动挂载列表移除（下次开机不再自动创建）    │
      │                                                        │
      │                                    [ 取消 ] [ ⏹ 卸载 ] │
      └────────────────────────────────────────────────────────┘
```

### 2.5 性能监控面板 (Performance)
借鉴 Windows 任务管理器，集成 LiveCharts2 绘制平滑的实时折线图。

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ ≡  VMem RAM Disk                                                 _ □ X   │
├────────┬─────────────────────────────────────────────────────────────────┤
│ 🏠 仪表盘 │  Performance                                                    │
│ 📊 性能   │  选择磁盘: [ R: Browser Cache ▼ ]                               │
│ ⚙️ 设置   │                                                                 │
│          │  全局物理内存总览:                                               │
│          │  [████████████████████▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░]           │
│          │   系统已用(20G)       RAM盘(5G)   可用(7G)                   │
│          │                                                                 │
│          │  I/O 吞吐量 (MB/s)                                              │
│          │  500 ┤     ╭╮                                                   │
│          │  400 ┤    ╭╯╰╮      ╭─╮                                         │
│          │  300 ┤   ╭╯  ╰╮    ╭╯ ╰╮                                        │
│          │  200 ┤  ╭╯    ╰────╯   ╰╮       ╭───                            │
│          │  100 ┤ ╭╯               ╰───────╯                               │
│          │    0 └─┴────┴────┴────┴────┴────┴────┴────┴────┴────            │
│          │      60s       45s       30s       15s       0s                 │
│          │      ── 读取 (Read)      ── 写入 (Write)                        │
│ ℹ️ 关于   │                                                                 │
├────────┴─────────────────────────────────────────────────────────────────┤
│ 🟢 Service: Connected  |  System RAM: [██████████░░░░░] 64% (20G/32G)    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.4 系统托盘 (TrayIcon)
使用 `Hardcodet.NotifyIcon.Wpf` 实现后台静默运行。

- **行为**：点击主窗口关闭按钮时，默认隐藏到托盘而非退出（可在设置中更改）。双击托盘图标恢复主窗口。
- **现代化右键菜单**：
  - 快速挂载/卸载列表（带盘符图标）。
  - 📊 性能悬浮窗（可选：鼠标悬停托盘图标时显示迷你性能卡片）。
  - ⚙️ 打开主界面。
  - ❌ 完全退出（断开 IPC，但不卸载已挂载的磁盘）。

---

## 3. 异常场景 UX 设计

### 3.1 Service 断连/未运行

GUI 启动时尝试连接 IPC 管道。如果连接失败或断开，进入**降级模式**：

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ ≡  VMem RAM Disk                                                 _ □ X   │
├────────┬─────────────────────────────────────────────────────────────────┤
│ 🏠 仪表盘 │                                                                 │
│ 📊 性能   │  ┌──────────────────────────────────────────────────────────┐ │
│ ⚙️ 设置   │  │  ⚠ 后台服务未运行                                       │ │
│          │  │                                                          │ │
│          │  │  VMem 后台服务未启动或无法连接。                         │ │
│          │  │  无法创建、管理或监控 RAM 盘。                           │ │
│          │  │                                                          │ │
│          │  │  [🔄 重试连接]  [🛠 启动服务]  [📂 查看日志]              │ │
│          │  │                                                          │ │
│          │  │  可能的原因：                                            │ │
│          │  │  · 服务未安装 → 请运行安装程序                          │ │
│          │  │  · 服务被停止 → 点击"启动服务"                          │ │
│          │  │  · 权限不足 → 请以管理员身份运行                        │ │
│          │  └──────────────────────────────────────────────────────────┘ │
│          │                                                                 │
│ ℹ️ 关于   │                                                                 │
├────────┴─────────────────────────────────────────────────────────────────┤
│ 🔴 Service: Disconnected  |  [重试中... 第 3 次]                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**重连策略**：
- 自动重连：1s → 2s → 4s → 8s → 15s → 30s（指数退避，最大 30s 间隔）
- 状态栏实时显示重连状态和计数
- 重连成功后自动刷新仪表盘
- [启动服务] 按钮调用 `sc start VMem`（需要管理员权限，弹 UAC 提示）

### 3.2 创建磁盘进度指示

> **UX 审查意见**：`PreAllocate=true` 创建 16GB 磁盘时需要一次性分配所有物理页，可能耗时数秒。
> 如果没有进度反馈，用户会以为 GUI 假死。

创建磁盘时 GUI 显示**不可取消的进度对话框**：

```text
      ┌────────────────────────────────────────────────────────┐
      │ 正在创建 RAM 盘...                                      │
      ├────────────────────────────────────────────────────────┤
      │                                                        │
      │  R: Browser Cache (2.0 GB)                             │
      │                                                        │
      │  [████████████████░░░░░░░░░░░░░░░░░░░░] 42%            │
      │                                                        │
      │  分配内存中... 860 MB / 2048 MB                        │
      │  预计剩余时间: 3 秒                                    │
      │                                                        │
      └────────────────────────────────────────────────────────┘
```

**实现方式**：
- `CreateDiskRequest` 改为异步：Service 收到后立即返回 `requestId`
- Service 通过事件通道推送进度事件：`DiskCreatingProgress { RequestId, Percent, AllocatedMB, TotalMB }`
- GUI 订阅进度事件更新 ProgressBar
- 非 PreAllocate 模式下创建几乎瞬间完成，仍显示短暂的 indeterminate 进度条

### 3.3 多用户场景

> **UX 审查意见**：同一台机器上可能有多个用户登录（Remote Desktop / 快速用户切换）。
> RAM 盘是系统级资源（Service 进程以 LocalSystem 运行），多个用户同时打开 GUI 会看到相同的磁盘。

**设计决策**：

| 场景 | 行为 |
|------|------|
| 用户 A 创建磁盘 R: | 所有用户可见（盘符是系统级的） |
| 用户 B 打开 GUI | 看到 R: 已挂载，可以监控和查看 |
| 用户 B 尝试卸载 R: | **PipeSecurity 检查**：仅 Administrators 组和创建者可卸载 |
| 用户 B 尝试修改 R: 设置 | 同上：仅创建者和 Admin 可修改 |
| 两个 GUI 同时连接 | Service 支持多客户端并发连接（每个连接独立管道实例） |
| SDDL 设置为 `IU`（交互用户） | 仅当前活跃控制台会话的用户可读写文件 |

**DiskConfig 扩展**：增加 `CreatorSid` 字段记录创建者，卸载/修改时验证调用者身份。

---

## 4. 全局设置与通知

### 4.1 设置页 (SettingsPage) — 完整布局

采用 PowerToys 风格的分组式设置页面，每组使用 `Expander` 可折叠展开。

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ ≡  VMem RAM Disk                                                 _ □ X   │
├────────┬─────────────────────────────────────────────────────────────────┤
│ 🏠 仪表盘 │  Settings                                                       │
│ 📊 性能   │                                                                 │
│ ⚙️ 设置   │  ┌─ 常规 ──────────────────────────────────────────────────┐   │
│          │  │ 开机自启 GUI                                 [  开关  ] │   │
│          │  │ 关闭窗口时最小化到托盘                       [  开关  ] │   │
│          │  └────────────────────────────────────────────────────────┘   │
│          │                                                                 │
│          │  ┌─ 外观 ──────────────────────────────────────────────────┐   │
│          │  │ 主题模式           [ 浅色 ○ ] [ 暗色 ○ ] [跟随系统 ◉] │   │
│          │  └────────────────────────────────────────────────────────┘   │
│          │                                                                 │
│          │  ┌─ 自动挂载管理 ─────────────────────────────────────────┐   │
│          │  │ 以下磁盘将在系统启动时自动创建：                      │   │
│          │  │                                                        │   │
│          │  │  R: Browser Cache    2GB  4KB页  快照:开 │ [开关][编辑]│   │
│          │  │  T: System TEMP      4GB  4KB页  快照:关 │ [开关][编辑]│   │
│          │  │                                                        │   │
│          │  │  提示：在仪表盘创建磁盘时勾选"自动挂载"即可添加      │   │
│          │  └────────────────────────────────────────────────────────┘   │
│          │                                                                 │
│          │  ┌─ 内存安全 ─────────────────────────────────────────────┐   │
│          │  │ 系统内存告警阈值      [====■======] 90%                │   │
│          │  │ (当系统物理内存使用超过此值时推送警告通知)             │   │
│          │  └────────────────────────────────────────────────────────┘   │
│          │                                                                 │
│          │  ▶ 后台服务 (点击展开)                                         │
│          │  ▶ 日志与诊断 (点击展开)                                       │
│          │                                                                 │
│ ℹ️ 关于   │                                                                 │
├────────┴─────────────────────────────────────────────────────────────────┤
│ 🟢 Service: Connected  |  System RAM: [██████████░░░░░] 64% (20G/32G)    │
└──────────────────────────────────────────────────────────────────────────┘
```

**展开"后台服务"区域**：

```text
  ┌─ 后台服务 ─────────────────────────────────────────────────────────┐
  │ 服务状态:     🟢 运行中 (PID: 12345, 运行时间: 3h 25m)            │
  │ 服务版本:     VMem.Service v1.0.0                                  │
  │ WinFsp 版本:  2.0.23075                                            │
  │                                                                    │
  │ [🔄 重启服务]  [🛠 修复 WinFsp 驱动]  [📂 打开配置目录]            │
  │                                                                    │
  │ 配置文件位置:  C:\ProgramData\VMem\config.json                     │
  │ 快照存储目录:  C:\ProgramData\VMem\snapshots\                      │
  └────────────────────────────────────────────────────────────────────┘
```

**展开"日志与诊断"区域**：

```text
  ┌─ 日志与诊断 ───────────────────────────────────────────────────────┐
  │ Service 日志级别:  [ Information ▼ ]  (运行时热切换，无需重启)     │
  │ GUI 日志级别:      [ Information ▼ ]  (需重启 GUI 生效)            │
  │ 启用回放日志:      [  关闭  ]  (排错时开启，会占用大量磁盘)       │
  │ 日志保留天数:      [ 7 ] 天                                        │
  │                                                                    │
  │ [📂 打开日志目录]  [📋 导出诊断报告]                               │
  │                                                                    │
  │ 日志目录: C:\ProgramData\VMem\logs\                                │
  └────────────────────────────────────────────────────────────────────┘
```

**设置项 → 配置存储映射**：

| 设置区域 | 设置项 | 保存位置 | 保存方式 |
|---------|--------|---------|---------|
| 常规 | 开机自启 GUI | 注册表 `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | GUI 直接写注册表 |
| 常规 | 最小化到托盘 | `%LocalAppData%\VMem\gui-settings.json` | GUI 本地保存 |
| 外观 | 主题模式 | `%LocalAppData%\VMem\gui-settings.json` | GUI 本地保存 |
| 自动挂载管理 | 开关/编辑 | `%ProgramData%\VMem\config.json` | GUI → IPC → Service 保存 |
| 内存安全 | 告警阈值 | `%ProgramData%\VMem\config.json` | GUI → IPC `UpdateGlobalSettings` → Service 保存 |
| 后台服务 | 重启服务 | - | GUI 调用 `sc stop/start VMem` |
| 日志与诊断 | Service 日志级别 | `%ProgramData%\VMem\config.json` | GUI → IPC `UpdateGlobalSettings` → Serilog 热切换 |
| 日志与诊断 | GUI 日志级别 | `%LocalAppData%\VMem\gui-settings.json` | GUI 本地保存，重启生效 |
| 日志与诊断 | 回放日志/保留天数 | `%ProgramData%\VMem\config.json` | GUI → IPC → Service 保存 |

### 4.2 消息通知 (Snackbar)
操作反馈不使用弹窗（MessageBox），而是使用底部弹出的 `Snackbar`：
- 成功："RAM 盘 R: 创建成功"
- 失败："挂载失败：盘符已被占用"
- 警告："系统内存不足，请注意保存数据"
- 信息："R: 配置已保存，自动挂载已启用"
- 信息："快照已完成 (R: 12.3MB)"

---

## 5. 学习与参考项目

1. **[WPF UI (lepoco)](https://github.com/lepoco/wpfui)**：本项目 GUI 的基石，参考其 Demo App 的导航与卡片设计。
2. **[PowerToys](https://github.com/microsoft/PowerToys)**：微软官方开源工具，其设置界面的排版、Expander 的使用是 Windows 11 UX 的最佳实践。
3. **[galpt/temp](https://github.com/galpt/temp)**：参考其"一键创建"的极简交互逻辑和实时统计的展示方式。
4. **[Rufus](https://github.com/pbatard/rufus)**：虽然是 Win32 原生界面，但其将复杂参数隐藏在"高级选项"折叠面板中的做法非常值得借鉴。

---

## 6. 日志与可观测性（L3 范式补丁）

> 详见 [子方案 ⑦ - 日志与可观测性](./vmem_07_logging_observability.plan.md)

### 5.1 全局异常处理与结构化上报

WPF 应用注册全局异常处理器，捕获未处理异常并写入 JSON 日志：

```csharp
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        DispatcherUnhandledException += (s, args) =>
        {
            Log.Error(args.Exception, "GUI unhandled exception on UI thread");
            args.Handled = true; // 防止崩溃，显示 Snackbar 错误通知
        };

        AppDomain.CurrentDomain.UnhandledException += (s, args) =>
        {
            Log.Fatal((Exception)args.ExceptionObject,
                "GUI fatal unhandled exception, isTerminating={IsTerminating}",
                args.IsTerminating);
        };

        TaskScheduler.UnobservedTaskException += (s, args) =>
        {
            Log.Error(args.Exception, "GUI unobserved task exception");
            args.SetObserved();
        };
    }
}
```

### 5.2 用户操作审计日志

关键用户操作写入 `INF` 级别日志，便于 AI 理解用户意图和操作序列：

| 用户操作 | 日志内容 |
|---------|---------|
| 点击"创建 RAM 盘" | 预设/自定义模式, 盘符, 容量, 页大小 |
| 点击"卸载" | 盘符, 当前使用量 |
| 切换导航页 | 目标页名称 |
| 修改设置 | 设置项, 旧值→新值 |
| IPC 连接状态变化 | Connected/Disconnected, 重连次数 |
| 应用启动 | 版本号, 系统信息, 启动耗时 |
| 应用退出 | 正常/异常, 运行时长 |

### 5.3 IPC 客户端通信日志

```csharp
public class IpcClientService
{
    public async Task<IpcResponse> SendAsync(IpcMessage request)
    {
        var sw = Stopwatch.StartNew();
        _logger.Debug("IPC sending {Type} requestId={RequestId}", request.Type, request.RequestId);
        try
        {
            var response = await SendInternalAsync(request);
            _logger.Information("IPC {Type} completed in {Ms}ms success={Success}",
                request.Type, sw.Elapsed.TotalMilliseconds, response.Success);
            return response;
        }
        catch (Exception ex)
        {
            _logger.Error(ex, "IPC {Type} failed after {Ms}ms", request.Type, sw.Elapsed.TotalMilliseconds);
            throw;
        }
    }
}
```

### 5.4 GUI 日志输出配置

GUI 进程单独维护一份日志文件，与 Service 进程隔离：

| Sink | 路径 | 说明 |
|------|------|------|
| JSON 文件 | `%ProgramData%\VMem\logs\vmem-gui-YYYYMMDD.json` | AI 解析的主要数据源 |
| 文本文件 | `%ProgramData%\VMem\logs\vmem-gui-YYYYMMDD.log` | 人类排错 |

GUI 日志默认级别 `INF`，可通过设置页调整为 `DBG`（重启生效）。
