# Qt 6 Linux 打包实战：把 Fcitx 插件随应用分发

Qt Quick 客户端在开发机上能运行，往往是因为 Qt Kit 已经提供了运行库、QML 模块和平台插件。把可执行文件拷给另一台机器时，这些依赖需要有明确的来源。红厅客户端的例子还多了一层：Fcitx 桥接插件由 Qt 在运行时加载，单看客户端的 ELF 链接依赖，无法知道项目还要分发这个插件。故障现场见 [银河麒麟 Qt 6 中文输入法异常的诊断与修复](银河麒麟Qt6中文输入法异常的诊断与修复.md)。

## 从构建产物到发布归档

在兼容目标机的 Linux 构建环境中准备 Qt Kit 和 `patchelf`，把 `QT_ROOT` 指向客户端和插件共用的 Qt Kit。以下三条命令在红厅客户端仓库根目录运行：

```sh
cmake --preset release -DREDCONF_PACKAGE_LINUX=ON
cmake --build --preset release --target RedConfClient --parallel 4
cpack --config build/release/CPackRedConfLinux.cmake -B build/release/packages
```

| 阶段 | 实际发生的事 |
| --- | --- |
| 配置 | CMake 读取 `release` 预设，选择 Qt Kit。打包开关注册 Linux 安装规则，检查插件和 `patchelf`，并生成独立的 CPack 配置。 |
| 构建 | 编译链接 `RedConfClient`，同时生成部署阶段使用的 QML 导入扫描结果。Fcitx 插件在此前已为这套 Kit 单独构建。 |
| 归档 | CPack 执行 `RedConfRuntime` 组件的安装规则，把运行所需的文件收进临时安装目录，再用 `TGZ` 生成 `.tar.gz`。 |

项目只在 Linux 上显式开启 `REDCONF_PACKAGE_LINUX` 时启用这套规则。专用配置文件 `CPackRedConfLinux.cmake` 也将应用发布和依赖项目自己的 CPack 配置分开。麒麟 2 上的 x86-64 发布包主要包含：

```text
RedConfClient-0.1.0-linux-x86_64/
├── bin/
│   ├── RedConfClient
│   ├── config.json
│   └── qt.conf
├── lib/
│   ├── libQt6*.so*
│   └── libVoteServerCore.so
├── qml/
│   └── 应用用到的 QML 模块
├── plugins/
│   ├── platforms/libqxcb.so
│   └── platforminputcontexts/libfcitxplatforminputcontextplugin-qt6.so
└── licenses/
    └── fcitx-qt6.LICENSE
```

`config.json` 来自仓库的默认配置。表决 SDK 动态库仅在定义了对应构建目标时加入。X11 平台插件位于 `platforms/`，输入法桥接插件位于 `platforminputcontexts/`，Qt 按插件类型到相应子目录查找。

## 显式安装插件，并把它纳入依赖分析

项目默认从当前 Qt Kit 的 `plugins/platforminputcontexts/` 取已编译的插件。如果插件单独存放，在第一条配置命令后追加 `-DREDCONF_FCITX_QT6_PLUGIN=/绝对路径/插件.so`。配置时用 `patchelf --print-needed` 检查插件是否依赖 `libQt6Gui.so.6`，提前拦住误用的 Qt 5 插件。架构、Qt 小版本和私有接口兼容性仍要靠匹配的 Kit 和目标机验证来保证。

下面摘录项目的安装规则。变量由 `LinuxPackaging.cmake` 定义，其中 `fcitx_plugin_dir` 为 `plugins/platforminputcontexts`。

```cmake
install(FILES "${fcitx_plugin}"
    DESTINATION "${fcitx_plugin_dir}" RENAME "${fcitx_plugin_name}" COMPONENT RedConfRuntime)
install(FILES "${CMAKE_CURRENT_LIST_DIR}/fcitx-qt6.LICENSE"
    DESTINATION licenses COMPONENT RedConfRuntime)
```

插件文件入包之后，还要让部署工具分析它依赖的库。项目在 `qt_finalize_target(RedConfClient)` 后生成安装阶段执行的部署脚本。以下是核心片段：

```cmake
qt6_generate_deploy_script(
    TARGET RedConfClient
    OUTPUT_SCRIPT deploy_script
    CONTENT "
qt_deploy_qml_imports(TARGET RedConfClient PLUGINS_FOUND qml_plugins)
qt_deploy_runtime_dependencies(
    EXECUTABLE \"$<TARGET_FILE:RedConfClient>\"
    ADDITIONAL_MODULES \${qml_plugins} \"${fcitx_plugin}\"
    GENERATE_QT_CONF)
execute_process(
    COMMAND \"${REDCONF_PATCHELF}\" --set-rpath \"$ORIGIN/${fcitx_to_lib}\"
        \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${fcitx_plugin_dir}/${fcitx_plugin_name}\"
    COMMAND_ERROR_IS_FATAL ANY)
")
install(SCRIPT "${deploy_script}" COMPONENT RedConfRuntime)
```

`qt_deploy_qml_imports` 根据构建时的扫描结果复制 QML 模块，`PLUGINS_FOUND` 将其中的插件交给后续依赖分析。Fcitx 插件同样通过 `ADDITIONAL_MODULES` 进入分析范围，它本身由前面的 `install(FILES ...)` 复制。脚本末尾的 `patchelf` 修改临时安装目录中的插件副本，源 Qt Kit 里的文件保留原状。 [Qt 运行时依赖部署文档](https://doc.qt.io/qt-6.8/qt-deploy-runtime-dependencies.html)

## Qt 和 Linux 分别去哪里找文件

解压后的 `bin/qt.conf` 是一份 INI 格式的运行时配置，实际内容只有两行：

```ini
[Paths]
Prefix = ..
```

在这套 Linux 布局里，Qt 以可执行文件所在的 `bin/` 为基准解析 `Prefix`，因此 `..` 指向包根目录。`Plugins`、`QmlImports` 和 `Libraries` 分别采用默认的 `plugins`、`qml` 和 `lib`，再相对于包根目录解析。整包换个解压位置也能按同样的布局找到模块和插件。 [Qt 的 qt.conf 路径规则](https://doc.qt.io/qt-6.8/qt-conf.html)

Linux 动态链接器读取 ELF 中的运行库搜索路径。客户端的安装路径是 `$ORIGIN/../lib`，桥接插件的路径是 `$ORIGIN/../../lib`。这里的 `$ORIGIN` 分别以可执行文件和插件自身的目录为起点，所以两条路径最终都指向包内的 `lib/`。Qt 先按插件目录找到桥接文件，动态链接器再根据 ELF 的搜索规则解析桥接文件所需的共享库。

目标 Linux 桌面仍需提供 X11、XKB 等系统运行依赖，并在启动客户端的同一个桌面会话运行 Fcitx 4 和中文输入法。会话设置 `QT_IM_MODULE=fcitx` 后，Qt 会尝试选择 Fcitx 桥接插件。macOS 使用系统输入法，沿用项目原有的 macOS 构建方式。

## 解压后验证

进入可信发布包的解压根目录，先核对归档布局和库解析结果：

```sh
cat bin/qt.conf
ldd bin/RedConfClient
ldd plugins/platforminputcontexts/libfcitxplatforminputcontextplugin-qt6.so
readelf -d plugins/platforminputcontexts/libfcitxplatforminputcontextplugin-qt6.so |
    grep -E 'RPATH|RUNPATH'
```

检查 Qt 库是否解析到本次解压的 `lib/`，再在目标图形会话通过 `QT_DEBUG_PLUGINS=1` 查看插件实际从哪里加载。QML 导入路径可配合 `QML_IMPORT_TRACE=1` 检查。完整业务验收还要用解压后的客户端打开人员管理搜索框，实际切换输入法、确认候选词并完成上屏。

原案例在麒麟 2 生成并解压了 x86-64 归档，`ldd` 确认客户端和插件的 Qt 依赖落在包内，独立 Qt 测试程序也从解压目录成功加载了桥接插件和 QML 模块。缺少桥接插件时的配置失败也已验证。人员搜索框的中文上屏是在此前修复的客户端上完成的，归档验证期间现场客户端仍沿用原部署。 [原始打包与验证记录](codex://threads/01a1aeae-dd38-78c3-ad99-af44786f217f)
