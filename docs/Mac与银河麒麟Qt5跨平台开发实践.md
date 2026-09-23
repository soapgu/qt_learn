# Mac 与银河麒麟 Qt 5 跨平台开发实践

## 1. 文档目标

本文基于一次完整的 Qt Widgets 跨平台验证，记录如何在 Apple Silicon Mac 上开发和调试，再将同一份源码交付到银河麒麟 x86-64 主机原生编译、运行和验收。

重点不是某个 Hello World 项目的代码，而是可复用的工作方式：

- 如何选择跨版本兼容基线。
- 如何在不同 CPU 架构和操作系统上复用源码。
- 如何验证工具链、构建产物、动态库和图形界面。
- 如何区分代码问题、IDE 误报和系统运行库问题。

## 2. 实测环境

### 2.1 Mac 开发机

| 项目 | 实测值 |
| --- | --- |
| CPU 架构 | Apple Silicon `arm64` |
| macOS | 26.6.2，Build 25G83 |
| Xcode | 26.2，Build 17C52 |
| Apple Clang | 17.0.0 |
| Homebrew | 6.0.16，前缀 `/opt/homebrew` |
| Qt | Homebrew `qt@5` 5.15.19 |
| qmake | `/opt/homebrew/opt/qt@5/bin/qmake` |
| Qt Creator | 20.0.1 Universal |
| 调试器 | `/usr/bin/lldb` |

### 2.2 银河麒麟目标机

| 项目 | 实测值 |
| --- | --- |
| 主机 | `172.16.16.31` |
| 用户 | `shgbit` |
| 系统 | 银河麒麟桌面操作系统 V10 SP1 2403 |
| 发行标识 | `Kylin-Desktop-V10-SP1-2403-Release-20240430-x86_64` |
| 内核 | 5.4.18-110-generic |
| CPU 架构 | `x86_64` |
| Qt | 系统 Qt 5.12.12-kylin |
| qmake | `/usr/bin/qmake`，QMake 3.1 |
| GCC/G++ | 9.3.0-10kylin5k0.5 |
| 图形环境 | X11/UKUI |
| 项目目录 | `/home/shgbit/qt_quick_start` |

> 以上主机、用户、路径和版本均为本次实测值，不是通用强制配置。

## 3. 核心技术决策

### 3.1 以目标机最低 Qt 版本为兼容基线

Mac 使用 Qt 5.15.19，麒麟使用 Qt 5.12.12。共享源码必须只使用 Qt 5.12 已存在的 API，不能因为 Mac 端能够编译就默认目标机也支持。

建议在代码中增加编译期边界：

```cpp
#include <QtGlobal>

#if QT_VERSION < QT_VERSION_CHECK(5, 12, 0)
#error "Qt 5.12 or newer is required"
#endif

#if QT_VERSION >= QT_VERSION_CHECK(6, 0, 0)
#error "This target is validated only with Qt 5"
#endif
```

实测项目使用 Qt Widgets、qmake 和 C++11，不使用 `.ui`、QML、Qt WebEngine、第三方库或平台专属 API，以减少第一次跨平台验证的变量。

### 3.2 交付源码，不交付开发机二进制

Mac 产物是 Mach-O `arm64`，麒麟需要 ELF `x86-64`；两端 Qt 的库格式、ABI、图形后端和系统依赖也不同。因此正确交付模式是：

```text
Mac 编写和验证源码
        ↓
Git 提交并推送
        ↓
麒麟拉取相同源码
        ↓
麒麟 Qt 5.12 + GCC 9.3 原生编译
```

构建目录应当独立并被 Git 忽略，例如 `build-mac/`、`build-qtcreator/` 和 `build-kylin/`。

## 4. Mac 端环境搭建

### 4.1 安装并核对 Qt 5

```bash
brew install qt@5

/opt/homebrew/opt/qt@5/bin/qmake -v
file /opt/homebrew/opt/qt@5/bin/qmake
```

实测 `qmake -v` 输出 Qt 5.15.19，`qmake`、QtCore 和 QtWidgets 均包含 `arm64` 架构。

`qt@5` 在 Homebrew 中是 keg-only 包，优先在项目命令和 Kit 中使用稳定路径 `/opt/homebrew/opt/qt@5`，不必为了方便而覆盖全局 Qt 环境。

### 4.2 配置 Qt Creator Kit

在 `Preferences → Kits` 中配置：

- Qt Version：`/opt/homebrew/opt/qt@5/bin/qmake`。
- C 编译器：Xcode 工具链中的 `clang`。
- C++ 编译器：Xcode 工具链中的 `clang++`。
- Debugger：`/usr/bin/lldb`。
- ABI：Darwin `arm64`。
- Kit 名称：`Desktop Qt 5.15 Apple Silicon`。

Qt Creator 界面可能把 Xcode 自动检测的工具链显示为 `Apple Clang iOS (arm64)`。判断配置是否正确时，应检查编译器实际路径、Darwin arm64 ABI、Qt Version 和 qmake 使用的 `macx-clang` mkspec，不要只看显示名称。

## 5. Mac 端构建与验证

### 5.1 命令行构建

```bash
mkdir -p build-mac
cd build-mac
/opt/homebrew/opt/qt@5/bin/qmake ../qt_quick_start.pro
make -j"$(sysctl -n hw.logicalcpu)"
```

不要只以“编译成功”作为验收结论，还应检查产物架构和动态库：

```bash
file qt_quick_start.app/Contents/MacOS/qt_quick_start
otool -L qt_quick_start.app/Contents/MacOS/qt_quick_start
```

实测产物为 Mach-O 64-bit `arm64`，链接 QtWidgets、QtGui 和 QtCore 5.15.19。

### 5.2 运行与调试

```bash
open -n build-mac/qt_quick_start.app
```

验收时同时检查：

- 窗口能够显示，标题和内容正确。
- 关闭窗口后进程正常退出。
- Qt Creator 使用指定 Kit 可以重新构建。
- LLDB 能启动调试，停止后 Qt Creator 显示调试器已结束。
- Qt Creator 构建产物也是 `arm64` 并链接预期 Qt 库。

### 5.3 新版 macOS SDK 警告

Qt 5.15 的 qmake 在较新 macOS SDK 上可能警告该 SDK 超出官方测试范围。这个警告表示组合未被上游完整覆盖，不等于当前项目构建失败。

处理原则是：

1. 保留警告和环境版本记录。
2. 实际验证 qmake、编译、链接、启动和调试。
3. 检查产物架构和 Qt 动态库。
4. 不把“本次成功”扩大为“所有 Qt 5.15 功能均与新 SDK 兼容”。

## 6. SSH 长期维护配置

### 6.1 只安装公钥

Mac 已有 `~/.ssh/id_ed25519.pub` 时，执行：

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub shgbit@172.16.16.31
```

验证免密登录：

```bash
ssh -o BatchMode=yes shgbit@172.16.16.31 'whoami; uname -m'
```

预期输出包含：

```text
shgbit
x86_64
```

不要复制或传输 `~/.ssh/id_ed25519`；它是私钥。

### 6.2 使用固定 SSH 别名

在 Mac 的 `~/.ssh/config` 中增加：

```sshconfig
Host kylin-dev 172.16.16.31
    HostName 172.16.16.31
    User shgbit
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

设置权限并验证：

```bash
chmod 600 ~/.ssh/config
ssh kylin-dev 'whoami; uname -m'
```

本次曾出现公钥已安装、命令行也能以 `shgbit` 登录，但 Codex 连接仍失败的现象。日志中的实际错误是：

```text
guhui@172.16.16.31: Permission denied (publickey,password).
```

原因是连接器默认使用了 Mac 用户名 `guhui`，而目标机用户是 `shgbit`。在 SSH config 中明确 `User shgbit` 后即可消除这类歧义。终端显示的后量子密钥交换警告不是这次登录失败的原因。

## 7. 麒麟端原生构建

### 7.1 先核对环境

```bash
ssh kylin-dev

uname -m
cat /etc/.kyinfo
/usr/bin/qmake -v
gcc --version
g++ --version
```

不要假设当前 `PATH` 中的 qmake 就是预期版本。在同时存在系统 Qt 5 和用户 Qt 6 的机器上，构建 Qt 5 项目时显式使用 `/usr/bin/qmake`。

### 7.2 传输或拉取源码

长期维护优先使用 Git：

```bash
cd /home/shgbit
git clone <repository-url> qt_quick_start
```

少量文件的临时验证也可以用 `scp`，但只传输源码、`.pro` 和文档，不传输 Mac 构建目录。

### 7.3 使用独立目录构建

```bash
cd /home/shgbit/qt_quick_start
mkdir -p build-kylin
cd build-kylin
/usr/bin/qmake ../qt_quick_start.pro
make -j"$(nproc)"
```

检查产物：

```bash
file ./qt_quick_start
ldd ./qt_quick_start
```

实测结果为 ELF 64-bit `x86-64`，链接目标机的 `libQt5Widgets.so.5`、`libQt5Gui.so.5` 和 `libQt5Core.so.5`，没有缺失动态库。

## 8. 远程 GUI 运行验证

SSH 会话默认不一定连接到麒麟当前桌面显示服务。目标机已有 `shgbit` 的 X11/UKUI 桌面会话时，可在确认实际显示号和 Xauthority 文件后启动：

```bash
DISPLAY=:0 \
XAUTHORITY=/home/shgbit/.Xauthority \
/home/shgbit/qt_quick_start/build-kylin/qt_quick_start
```

不应盲目假设所有机器都使用 `:0` 或 `~/.Xauthority`。如果启动失败，先核对当前桌面会话的 `DISPLAY`、`XAUTHORITY`、用户和进程权限。

本次验收包括：

- 窗口在目标机桌面正常显示。
- 窗口尺寸为 320 × 180。
- `xdotool getwindowname` 返回 `Qt Quick Start`。
- 窗口中央显示 `Hello World`。
- 同一份 `main.cpp` 和 `.pro` 文件无需修改即可编译和启动。

远程图形启动的成功只能证明当前用户会话和环境变量可用；部署时仍应根据自启动、systemd 或现场登录方式单独设计运行环境。

## 9. Qt Creator 代码模型误报

麒麟端曾出现项目可以使用 qmake 和 GCC 成功编译，但编辑器仍标记 `unknown type name 'QApplication'` 的情况。这类现象不能直接认定为代码或 Qt 安装错误。

首先核对 Kit：

- Device：`Local PC`。
- C：`/usr/bin/gcc`。
- C++：`/usr/bin/g++`。
- Debugger：`/bin/gdb`。
- Qt：系统 Qt 5.12.12。

然后依次：

1. 执行 qmake。
2. 清理项目。
3. 重新构建。
4. 重新打开报错源文件。
5. 让 Qt Creator 重新解析外部变更文件。

如果实际编译仍然成功，但编辑器报错不消失，问题属于 Qt Creator 4.11 的 Clang Code Model 误报的可能性很高。可将禁用 `ClangCodeModel` 插件作为隔离验证，而不是为了消除红线去修改本来可正常编译的业务代码。

## 10. 麒麟退出阶段 SIGTRAP

### 10.1 现象

麒麟端窗口可正常显示和响应，但关闭窗口时进程在 Qt 5.12.12-kylin 的 XCB/GLib 清理流程中收到 `SIGTRAP`：

```text
GLib-GIO-ERROR: g_settings_schema_source_unref() called too many times on the default schema source
```

`coredumpctl` 显示调用栈集中在：

- `libglib-2.0.so.0`
- `libQt5XcbQpa.so.5`
- `libQt5Gui.so.5`

### 10.2 已完成的隔离验证

- 相同源码在 Mac 可正常关闭。
- 麒麟端可正常编译、链接、启动和显示窗口。
- `file` 和 `ldd` 检查正常，没有缺失 Qt 动态库。
- 禁用平台主题后仍可复现。
- 设置 `QT_NO_GLIB=1` 后仍可复现。

因此，当前证据更指向目标机的 Qt 5.12.12-kylin、XCB 平台插件与 GLib/GSettings 清理阶段的兼容问题，而不是项目的编译或主界面逻辑失败。

### 10.3 处理边界

本次验证没有宣称该问题已解决。正确记录方式是：

- 将“编译成功”、“窗口显示成功”与“程序正常退出”分为三个验收项。
- 保留 core dump、完整错误文本、Qt/GLib 包版本和调用栈。
- 用麒麟官方更新、纯净用户环境或最小 Qt Widgets 程序继续复现。
- 不混入其他 Linux 发行版的 Qt、GLib 或 XCB 包来“覆盖”系统库。
- 在根因解决前，明确将正常退出标记为未通过项。

## 11. 可复用验收清单

### 11.1 Mac 端

- [ ] `qmake -v` 显示预期 Qt 5.15 版本。
- [ ] qmake 和 Qt 核心库包含 `arm64`。
- [ ] 命令行和 Qt Creator 都能构建。
- [ ] 窗口启动、内容显示和正常退出通过。
- [ ] LLDB 调试启动和停止通过。
- [ ] `file` 显示 Mach-O `arm64`。
- [ ] `otool -L` 显示预期 Qt 5.15 库。

### 11.2 麒麟端

- [ ] SSH 别名使用正确用户且免密连接。
- [ ] `uname -m` 显示 `x86_64`。
- [ ] `/usr/bin/qmake -v` 显示 Qt 5.12.12-kylin。
- [ ] GCC/G++ 版本与目标工具链一致。
- [ ] 只使用源码在麒麟独立构建目录中重新编译。
- [ ] `file` 显示 ELF `x86-64`。
- [ ] `ldd` 无缺失库且链接系统 Qt 5.12。
- [ ] 窗口在实际 X11/UKUI 桌面显示正常。
- [ ] 窗口标题、内容和交互符合预期。
- [ ] 关闭窗口后进程正常退出，无 core dump。

## 12. 经验总结

1. **兼容性由最低版本决定。** 开发机 Qt 更新不代表可使用新 API。
2. **跨平台交付的核心是源码可复现构建。** 不要在 macOS arm64 和 Linux x86-64 之间复制二进制产物。
3. **IDE 诊断不是编译器结论。** 代码模型报错时，先用实际 qmake 和编译命令确认真假。
4. **SSH 失败要看实际登录用户。** 公钥正确不代表连接器使用了正确用户名。
5. **GUI 启动是独立验收阶段。** 编译通过不能证明 X11、平台插件、字体和桌面会话正常。
6. **启动正常不代表退出正常。** 启动、运行和退出必须分别验收。
7. **故障记录必须区分证据和推断。** 保留命令输出、动态库、调用栈和已试过的隔离变量，未解决问题不应标记为已完成。
