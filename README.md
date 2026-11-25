
懒得两端推了，一起放gitee吧：https://gitee.com/caitangbutian/20261030---mouse-side-button

# mouse-side-keys-rs

用鼠标侧键切换 Windows 虚拟桌面的小工具（Rust 版）。

侧键按下时被转换为 `Ctrl + Win + ← / →`，由 Windows 自身完成桌面切换。程序只依赖 Win32 API，编译产物是**单个原生 exe**，不需要 Python、.NET、VC++ 运行库或其他运行时。

## 1. 它是做什么的

| 操作 | 默认映射（方向已对调） | `--no-swap` 标准映射 |
| --- | --- | --- |
| 侧键 1（后退键 / XBUTTON1） | `Ctrl + Win + →`（右侧桌面） | `Ctrl + Win + ←`（左侧桌面） |
| 侧键 2（前进键 / XBUTTON2） | `Ctrl + Win + ←`（左侧桌面） | `Ctrl + Win + →`（右侧桌面） |

主要特性：

- **零运行时、小体积**：`windows-sys` 只做链接期绑定；release 配置开启 `opt-level="z"`、LTO、剥符号、`panic=abort`，产物约 270 KiB。
- **单文件、无安装**：exe 拷走即用，内置图标。
- **可见界面、无托盘**：主窗口显示运行状态和启动/停止按钮；运行中关窗口转后台，已停止时关窗口直接退出。
- **单实例接管**：双击 exe 会结束同名旧进程并弹全新界面（后台实例也会被接管）。
- **权限自适应**：默认自动请求 UAC 提权，解决高权限窗口聚焦时侧键失效的问题（UIPI）。
- **可容错**：内置多种按键合成预设，可在现场自检哪种方式可用。

## 2. `mouse-side-keys-rs.exe` 是编译产物吗

**是。** 它不是源码，也不是脚本，而是 `cargo build --release` 生成的 Windows 原生可执行文件：

- 本地这个文件：PE32+ / x64（Machine `0x8664`）、GUI 子系统（Subsystem `2`）、6 个节、279,040 字节、PE 时间戳 `2026-10-31 12:57`（北京时间）。
- 源码在 `src/main.rs`；构建配置在 `Cargo.toml`；图标来自 `mouse.ico`，由 `build.rs` 在 Windows 构建时嵌入。
- 仓库里的 `master` 分支不跟踪该 exe，它由 GitHub Actions 构建后提交到 **`build-output` 分支**，本地文件是从那里取回的现成二进制。

不要手改这个 exe，也不要把它当源码维护；需要变更时改 `src/main.rs` 后重新构建。

### 取回已编译的 exe

公开仓库可直接用 raw 链接匿名下载 `build-output` 分支上的 `mouse-side-keys-rs.exe`；若本地已配置名为 `github` 的 remote，也可在仓库根目录执行：

```bash
git fetch github build-output
git show github/build-output:mouse-side-keys-rs.exe > mouse-side-keys-rs/mouse-side-keys-rs.exe
```

同一分支上还有 `build.log`，可用于核对编译过程。

### 自己编译

环境要求：

- Windows x86_64（程序直接调用 Win32 API，目前仅支持 Windows）
- 稳定版 Rust 工具链
- MSVC 链接器：Visual Studio Build Tools 的「使用 C++ 的桌面开发」工作负载

在本目录执行：

```bash
cargo build --release
```

产物位于 `target/release/mouse-side-keys-rs.exe`。把源码拷到别的机器编译时，记得连同 `mouse.ico`、`build.rs`、`.cargo/config.toml` 一起带上。

### CI 自动构建

`.github/workflows/build.yml` 在推送到 `master` 或手动触发 `workflow_dispatch` 时运行：

1. `windows-latest` 上安装 stable Rust；
2. `cargo build --release`，日志写入 `build.log`；
3. 校验 `target/release/mouse-side-keys-rs.exe` 存在；
4. 把 exe 和 `build.log` 强制提交到 `build-output` 分支。

## 3. 怎么用（普通用户）

1. 双击 `mouse-side-keys-rs.exe`。首次运行会弹 UAC，**点“是”**，否则侧键可能被系统拦截。
2. 主窗口显示 `状态： 已停止`，点 **“启动”** 按钮后变成 `状态： 运行中`，侧键立即生效。
3. 退出方式：
   - 运行中关闭窗口：**不退出**，窗口消失但钩子继续在后台工作；
   - 已停止时关闭窗口：**退出程序**；
   - 随时可用命令行 `mouse-side-keys-rs.exe --stop` 结束进程。
4. 后台运行后想再操作：**再次双击 exe**。新进程会结束旧的后台实例并弹出全新界面。
5. 勾选 **“开机自动启动（管理员权限运行）”**：注册名为 `MouseSideKeys` 的计划任务（`ONLOGON` + `HIGHEST`），登录时以后台静默模式启动，不弹 UAC、不显示界面。取消勾选即删除该任务。此操作需要管理员权限；笔记本会同时关闭“只在交流电源时启动”的限制。

> 注意：单实例接管是按 **exe 文件名** 匹配进程的。两个不同目录下都叫 `mouse-side-keys-rs.exe` 的程序会互相结束，避免同时装两个实例。

## 4. 命令行参数（开发人员）

从 `cmd` 或终端启动时，程序会附加到父控制台输出信息；直接双击时，命令模式的输出会汇总到一个消息框显示。

```bat
mouse-side-keys-rs.exe                    :: 界面模式（默认）
mouse-side-keys-rs.exe --no-swap          :: 标准方向：侧键1向左、侧键2向右
mouse-side-keys-rs.exe --no-block         :: 不吞掉侧键原有的前进/后退功能
mouse-side-keys-rs.exe --debounce 150     :: 去抖毫秒，默认 80，0 为关闭
mouse-side-keys-rs.exe --test             :: 自检：发一次 Ctrl+Win+→ 并逐个尝试预设
mouse-side-keys-rs.exe --list-presets     :: 列出按键合成预设
mouse-side-keys-rs.exe --preset rawsplit  :: 指定预设（默认 rawsplit）
mouse-side-keys-rs.exe --gap 20           :: 修饰键之间的间隔（兜底参数）
mouse-side-keys-rs.exe --split 20         :: 每个输入事件单独发送，间隔毫秒
mouse-side-keys-rs.exe --no-raw           :: 强制使用虚拟键码而非扫描码
mouse-side-keys-rs.exe --stop             :: 按 PID 文件结束运行中的实例
mouse-side-keys-rs.exe --elevate          :: 请求 UAC 以管理员身份重启
```

| 参数 | 说明 |
| --- | --- |
| 无参数 | 显示主界面；功能默认不启动，需点“启动” |
| `--background` | 内部使用：由开机自启的计划任务传入，无界面、无托盘，直接装钩子 |
| `--test` | 打印本进程完整性级别，依次用各预设发送一次 `Ctrl+Win+→`，提示哪个可用 |
| `--list-presets` | 列出所有预设及说明 |
| `--preset <name>` | 选择预设：`rawsplit`（默认）、`raw`、`split`、`no-ext`、`default` |
| `--gap N` / `--split N` | 覆盖预设里的间隔参数，可与 `--preset`、`--no-raw` 自由组合 |
| `--no-raw` | 用虚拟键码发送（默认用扫描码） |
| `--debounce N` | 连击去抖窗口，默认 80ms，`0` 关闭 |
| `--no-swap` / `--swap` | 标准方向 / 对调方向（默认对调） |
| `--no-block` | 保留浏览器等程序里侧键原有的前进/后退功能 |
| `--stop` | 读取 `%TEMP%\mouse_side_keys_rs.pid` 并 `taskkill` 对应进程 |
| `--elevate` / `--no-elevate` | 手动请求提权 / 跳过自动提权（调试用） |
| `--as-admin` | 内部使用：提权重启后防止无限请求 UAC |
| `--foreground` | 兼容旧版参数，当前已被解析但不改变运行方式；从终端启动即可看到输出 |

### 预设怎么选

| 预设 | 方式 |
| --- | --- |
| `rawsplit` | **默认**。扫描码（`wVk=0`）+ 每个事件单独发送 + 10ms 间隔 |
| `raw` | 扫描码，整批发送 |
| `split` | 虚拟键码，逐个发送 |
| `no-ext` | 扫描码但不带 `KEYEVENTF_EXTENDEDKEY` |
| `default` | 虚拟键码整批发送（常被过滤拒绝） |

本机实测只有“扫描码 + 逐个发送 + 间隔 ≥10ms”这一种组合能同时绕过「Win 键虚拟键码被过滤」和「整批 `SendInput` 被拒」两层限制，所以默认用 `rawsplit`。排查参数时务必保持可自由组合，不要做成互斥枚举。

## 5. 排障

- **侧键完全没反应**：从终端运行 `mouse-side-keys-rs.exe --test`，看输出的完整性级别。不是 `High (0x3000)` 就先用 `--elevate` 提权；若提示 UIPI 警告，说明前台有更高权限的窗口（如以管理员运行的 360），提权后即可恢复。
- **提示“按键被系统拦截”弹窗**：这是 UIPI 检测的结果，每次会话只弹一次。弹窗跑在独立线程，不会阻塞持有低级钩子的主循环（否则鼠标会整体卡顿）。
- **所有预设都失败**：安全软件可能拦截了合成输入，把本目录加入白名单后重试。
- **`--stop` 提示缺少 PID 文件**：程序未运行，或 PID 文件（`%TEMP%\mouse_side_keys_rs.pid`）已被清理；直接双击 exe 会接管同名进程，也可在任务管理器结束它。
- **开机自启失败**：注册计划任务需要管理员权限；若任务是旧版本注册的（命令行没有 `--background`），程序启动时会自动用静默参数重新注册。
- **移动 exe 位置后开机自启失效**：计划任务记录的是旧绝对路径，需要取消勾选后重新勾选。

## 6. 实现要点（改代码前必读）

1. **钩子线程不能做重活**：`WH_MOUSE_LL` 回调只调用 `PostThreadMessageW(WM_APP_TRIGGER)`；真正的 `SendInput` 在主消息循环里执行。任何模态弹窗或阻塞调用都不能放在持有钩子的线程上。
2. **按键合成三要素**：`KEYEVENTF_SCANCODE`（`wVk=0`）+ 逐个事件发送 + 事件间隔 ≥10ms。破坏任意一条都可能导致静默失败（`SendInput` 返回 0 且 `GetLastError=0`）。
3. **GUI 子系统设置方式**：在 `src/main.rs` 顶部用 `#![windows_subsystem = "windows"]`。不要改回 `.cargo/config.toml` 的 `rustflags=/SUBSYSTEM:WINDOWS`，那会误伤依赖的 build script，触发 `LNK2019: unresolved external symbol WinMain`。`.cargo/config.toml` 刻意保持为空配置。
4. **windows-sys 0.52 的句柄是 `isize` 不是指针**：空句柄判断用 `== 0`，不要用 `null_mut()` / `.is_null()`。
5. **部分 Win32 函数手写 FFI**：`OpenProcessToken`、`GetModuleHandleW` 等在 windows-sys 0.52 中的模块位置与预期不符，直接 `unsafe extern "system"` 声明更省事。
6. **单实例接管要留退出时间**：`TerminateProcess` 后固定等约 300ms，让旧实例卸载钩子、销毁窗口。
7. **回归测试要断言具体取值**：方向映射、默认预设等必须断言实际常量值，不能只断言“没报错”。

## 7. 目录结构

```
mouse-side-keys-rs/
├── Cargo.toml                  # 依赖与 release 体积优化配置
├── build.rs                    # Windows 下把 mouse.ico 嵌入 exe 资源
├── mouse.ico                   # 程序图标
├── mouse-side-keys-rs.exe      # 编译产物（来自 build-output 分支）
├── src/main.rs                 # 全部源码：Win32 钩子、按键合成、界面、参数解析
├── .cargo/config.toml          # 刻意为空，见实现要点 3
└── .github/workflows/build.yml # CI：构建并发布 exe 到 build-output 分支
```

同一仓库根目录的 `mouse_side_keys.py` 是本工具的 Python 前身，逻辑行为基本一致；本目录是完全独立的 Rust 重写版，运行时互不影响。
