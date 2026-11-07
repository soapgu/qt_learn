# 银河麒麟 Qt 6 中文输入法异常的诊断与修复

在银河麒麟 V10 SP1 的 X11 桌面上，红厅客户端的「人员管理」搜索框无法切换中文，Fcitx 状态提示「无输入窗口」。这台机器上显示的中文输入法是搜狗输入法，桌面会话运行的输入法框架是 Fcitx 4，客户端则使用一套独立安装的 Qt 6.8.3。

## Fcitx 是什么

Fcitx（常称「小企鹅输入法」）是 Linux 桌面上的输入法框架。它管理可用的输入法，并在应用和当前输入法之间传递输入事件与上屏文字。Fcitx 4 和 Fcitx 5 是这个框架的不同主要版本，本案例的桌面会话运行的是 Fcitx 4。 [Fcitx 项目文档](https://fcitx-im.org/wiki/Fcitx)

以在红厅客户端键入 `nihao` 为例，Qt 6 通过桥接插件把输入交给 Fcitx 4，当前选用的搜狗输入法负责生成拼音候选词。选中「你好」后，文字沿同一条链路回到 Qt 输入框。桌面上显示「搜狗输入法」，是因为它是用户当前选用的输入法。 [搜狗输入法 Linux 安装指南](https://pinyin.sogou.com/linux/help.php)

```text
Qt 6 输入框 ↔ Qt 6 桥接插件 ↔ Fcitx 4 框架 ↔ 搜狗输入法
```

## 从输入框查到桌面会话

先确认人员管理窗口已经在前台，并在搜索框里复现问题。「无输入窗口」是 Fcitx 对当前输入上下文的判断。即便窗口已经在前台，仍要确认应用有没有通过 Qt 的输入法桥接插件向 Fcitx 报告输入焦点。

通过 SSH 检查进程时，关键是读取**正在运行的 GUI 进程**的环境。SSH shell 自己的环境代表远程终端，和启动客户端的桌面会话可能不同。下面的命令可以在 Linux 上复用，先从进程列表里选出当前桌面对应的两个 PID。

```sh
pgrep -a -x RedConfClient
pgrep -a -x fcitx

app_pid=1234    # 替换成客户端 PID
fcitx_pid=5678  # 替换成 Fcitx PID
for pid in "$app_pid" "$fcitx_pid"; do
    printf '\nPID %s\n' "$pid"
    tr '\000' '\n' < "/proc/$pid/environ" |
        grep -E '^(DISPLAY|DBUS_SESSION_BUS_ADDRESS|QT_IM_MODULE)='
done
```

现场确认 Fcitx 4 正在运行，客户端和 Fcitx 处于同一个 X11 桌面及 D-Bus 会话，客户端进程已经带有 `QT_IM_MODULE=fcitx`。接下来要验证的是 Qt 能否实际找到并加载这个输入法模块。 [Fcitx 排查说明](https://fcitx-im.org/wiki/FAQ/en)

## 用实际加载的库缩小范围

Linux 的 `/proc/<PID>/maps` 可以用来查看共享库当前是否已映射进进程。以同桌面上可使用中文输入法的 Qt 5 程序作为对照，在两个程序的输入框各复现一次，再分别检查：

```sh
qt5_pid=9012  # 替换成对照程序 PID
for pid in "$app_pid" "$qt5_pid"; do
    printf '\nPID %s\n' "$pid"
    grep -E 'libQt[56]Gui|/platforminputcontexts/' "/proc/$pid/maps" |
        awk '{print $NF}' | sort -u
done
```

现场看到，系统 Qt 5 程序加载了系统的 Fcitx 桥接插件。红厅客户端使用独立的 Qt 6.8.3，输入法插件里只看到 `compose`，而这套 Qt 的插件目录缺少 Fcitx 的 Qt 6 桥接文件。系统 Qt 5 程序能输入中文，说明桌面服务可用。红厅客户端的进程证据则把问题定位到了 Qt 6 的输入法接入层。

如果只看进程映射，能确认当前加载了什么。想进一步区分插件目录里没有文件、Qt 发现了文件但拒绝加载，以及插件依赖缺失，可以在同一个图形桌面会话里启动一个用于排查的实例：

```sh
QT_DEBUG_PLUGINS=1 QT_IM_MODULE=fcitx /path/to/RedConfClient \
    > qt-plugins.log 2>&1
grep -Ei 'platforminputcontexts|fcitx|compose|cannot load' qt-plugins.log
```

先看 Qt 搜索了哪些目录，再区分「找到插件元数据」和「成功加载库」两类日志。若出现加载失败，可对可信插件运行 `ldd` 检查缺失依赖。 [Qt 插件部署与诊断](https://doc.qt.io/qt-6.8/deployment-plugins.html)

## 给客户端补上 Qt 6 桥接插件

Fcitx 4 的 Qt 前端来自上游 `fcitx-qt5` 项目。案例使用的 1.2.7 也能构建 Qt 6 插件。用客户端所用的 Linux Qt 6.8.3 Kit 在兼容目标机的环境里构建。下面的命令假定已经解压源码、安装上游构建依赖，并把 `QT_ROOT` 设置为该 Kit 的根目录。

```sh
cmake -S /path/to/fcitx-qt5-1.2.7 -B build-fcitx-qt6 \
    -DCMAKE_PREFIX_PATH="$QT_ROOT" \
    -DENABLE_QT6=ON -DENABLE_QT5=OFF -DENABLE_LIBRARY=OFF
cmake --build build-fcitx-qt6 --parallel 4
find build-fcitx-qt6 -name 'libfcitxplatforminputcontextplugin-qt6.so'
```

将找到的 `libfcitxplatforminputcontextplugin-qt6.so` 安装到客户端所用 Qt 插件根目录下的 `platforminputcontexts/`。本次现场修复使用的是 Qt Kit 的 `plugins/platforminputcontexts/`。在原桌面会话重启客户端，让 Qt 在 `QT_IM_MODULE=fcitx` 的配置下重新选择输入模块。桥接插件负责和桌面里的 Fcitx 4 通信，拼音转换和候选词由桌面端的中文输入法处理。 [Fcitx Qt 前端源码发布目录](https://download.fcitx-im.org/fcitx-qt5/)

现场在人员管理搜索框按 `Ctrl+空格` 切到中文，键入 `nihao` 后成功上屏「你好」。本次实际操作验证覆盖人员搜索框，测试文字随后清空。修复落在 Qt 运行环境，人员管理界面的 QML 和业务逻辑沿用原实现。 [原始诊断记录](codex://threads/01a1aeae-dd38-78c3-ad99-af44786f217f)

要把这次修复带到其他机器，可以在发布时把桥接插件和 Qt 运行库一起收进归档。完整过程见 [Qt 6 Linux 打包实战](Qt6%20Linux打包实战与Fcitx插件分发.md)。
