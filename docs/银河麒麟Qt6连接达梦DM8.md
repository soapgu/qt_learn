# 银河麒麟 Qt 6 连接达梦 DM8：从 CMake 到 ODBC 部署

给签到客户端加上真实数据源后，Qt 程序需要定期从达梦 DM8 读取签到状态。接入分成两步。先在工程里编译并链接 Qt SQL，再准备运行时的 QODBC 插件、unixODBC 和达梦驱动。只完成第一步，程序可以编译通过，却可能在打开数据库时失败。

本文以签到客户端提交 `da0cb20` 为例。历史实测环境为银河麒麟 V10 SP3 x86_64、Qt 6.8.3，以及达梦 DM8 的 2026-07-10 银河麒麟 V10 SP3 x86_64 客户端包。

## Qt SQL 如何找到达梦

```text
客户端的 QSqlDatabase / QSqlQuery
    → Qt 的 QODBC 插件
    → 系统的 unixODBC 驱动管理器
    → 达梦 ODBC 驱动 libdodbc.so
    → DM8 服务
```

| 依赖 | 工程如何使用 |
| --- | --- |
| `Qt6::Sql` | CMake 链接到实现数据库查询的目标，C++ 代码使用 `QSqlDatabase` 和 `QSqlQuery`。 |
| `libqsqlodbc.so` | Qt 运行时从 `sqldrivers` 插件目录加载。构建环境使用匹配的 Qt Kit，发布后在安装包的 `plugins/sqldrivers/` 核对该插件。 |
| unixODBC | 由麒麟系统提供运行库。它根据驱动注册信息找到达梦驱动。 |
| `libdodbc.so`、`libdmdpi.so`、`libdmfldr.so` | 来自 DM8 客户端包的 `drivers/odbc`。本项目获授权后纳入 `vendor/dameng-odbc/lib/linux-x86_64/`，随应用分发。 |

应用通过 Qt SQL 接口发起查询，因此无需引入达梦 SDK 头文件，也无需在 `target_link_libraries()` 里直接链接这三个达梦库。Qt 的驱动插件机制和达梦在 Linux 上的驱动注册方式见官方文档。[Qt SQL 驱动文档](https://doc.qt.io/qt-6/sql-driver.html) · [达梦 ODBC 配置指南](https://eco.dameng.com/community/training/d78cfb7003f655efd997a60645c43759)

## 在 CMake 里加入构建与部署依赖

模拟签到源一直参与构建。工程只在 Linux x86_64 上启用真实签到源，下面是从项目摘出的核心配置。

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux"
   AND CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
    set(REDCONF_HAS_DAMENG_ODBC ON)
    find_package(Qt6 6.8.3 REQUIRED COMPONENTS Sql)
endif()

if(REDCONF_HAS_DAMENG_ODBC)
    target_sources(RedConfCheckInSources PRIVATE
        infrastructure/DmCheckInSource.h
        infrastructure/DmCheckInSource.cpp)
    target_compile_definitions(RedConfCheckInSources PRIVATE REDCONF_HAS_DAMENG_ODBC)
    target_link_libraries(RedConfCheckInSources PRIVATE
        Qt6::Sql Qt6::Concurrent RedConfCheckInCredentials)
endif()
```

打包也使用同一个条件。构建时让 `RedConfClient` 依赖复制任务，使用 `cmake -E copy_if_different` 把三个驱动库放进可执行文件旁的 `dameng-odbc/`，同时把启动脚本放到可执行文件旁。安装规则沿用相同布局，Qt 的部署脚本负责收集 Qt 运行库和插件。

```cmake
if(REDCONF_HAS_DAMENG_ODBC)
    set(driver_dir "${PROJECT_SOURCE_DIR}/vendor/dameng-odbc/lib/linux-x86_64")
    set(redconf_dameng_libraries
        "${driver_dir}/libdodbc.so"
        "${driver_dir}/libdmdpi.so"
        "${driver_dir}/libdmfldr.so")
    set(redconf_linux_launcher "${PROJECT_SOURCE_DIR}/scripts/run-redconf-linux.sh")
    set(client_dir "$<TARGET_FILE_DIR:RedConfClient>")

    add_custom_target(RedConfDmOdbcRuntime
        COMMAND ${CMAKE_COMMAND} -E make_directory "${client_dir}/dameng-odbc"
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${redconf_dameng_libraries} "${client_dir}/dameng-odbc"
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${redconf_linux_launcher}" "${client_dir}"
        VERBATIM)
    add_dependencies(RedConfClient RedConfDmOdbcRuntime)
endif()
```

```cmake
if(REDCONF_HAS_DAMENG_ODBC)
    install(PROGRAMS "${redconf_linux_launcher}"
        DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT RedConfRuntime)
    install(FILES ${redconf_dameng_libraries}
        DESTINATION "${CMAKE_INSTALL_BINDIR}/dameng-odbc" COMPONENT RedConfRuntime)
endif()
```

项目同时安装供应商许可文件。Qt 部署和发布包的构建方式见[《Qt 6 Linux 打包实战》](Qt6%20Linux打包实战与Fcitx插件分发.md)。

macOS 上，这个项目暂未接入达梦 ODBC 驱动，日常开发使用模拟签到。真实签到源和 Linux 驱动库只进入上述 Linux x86_64 分支。

## 配置地址，再由启动脚本注册驱动

先编辑客户端旁的 `config.json`。下面只列出需要调整的签到字段，地址为示例。

```json
{
  "checkin_mock": false,
  "checkin_connection_string": "192.0.2.10:5236"
}
```

`checkin_connection_string` 只接受主机名或 IPv4 地址加端口。以应用运行用户进入安装目录，在终端交互输入只读账号和密码，工具会把 `checkin-credentials.enc` 写到 `config.json` 所在目录。

```sh
./bin/RedConfCheckInCredentials ./bin
./bin/run-redconf-linux.sh
```

仓库没有提交 `.ini` 文件，使用者也无需手工创建 `odbc.ini` 或配置 DSN。启动脚本会在当前用户的运行目录或缓存目录生成 `odbcinst.ini`，内容只有驱动名和实际驱动路径。

```ini
[DM8 ODBC DRIVER]
Driver=/path/to/app/bin/dameng-odbc/libdodbc.so
```

脚本把 `ODBCSYSINI` 指向生成文件所在目录，并设定 `ODBCINSTINI=odbcinst.ini`。它还将驱动目录和兼容库目录加到本次进程的 `LD_LIBRARY_PATH`。这一版达梦驱动会查找未带版本号的 `libodbcinst.so` 和 `libcrypto.so`，脚本在自己的运行目录中分别把它们链接到系统的 `libodbcinst.so.2` 和 `libcrypto.so.1.1`。应用自身如何链接加密库，见[《银河麒麟 Qt 项目接入 OpenSSL》](银河麒麟Qt项目接入OpenSSL构建AES.md)。

## 在系统级注册达梦 ODBC 驱动

如果麒麟机器由管理员统一安装达梦客户端，可以把驱动注册到系统级 `odbcinst.ini`，供同一台机器上读取这份配置的 ODBC 程序使用。`odbcinst.ini` 把驱动名映射到动态库路径，`odbc.ini` 则用来定义命名数据源（DSN）。本项目直接在连接串里指定驱动名和连接参数，因此系统级注册也可以免 DSN。[unixODBC 驱动配置说明](https://www.unixodbc.org/odbcinst.html)

先查看这台机器的 unixODBC 使用哪个系统级驱动配置文件。输出中的 `DRIVERS` 是驱动注册文件路径，通常为 `/etc/odbcinst.ini`，实际位置以命令输出为准。

```sh
env -u ODBCSYSINI -u ODBCINSTINI odbcinst -j
```

由管理员在该文件中添加以下条目，保留已有的其他驱动。`Driver` 使用已安装的达梦驱动的实际绝对路径，节名保持为 Qt 连接串使用的 `DM8 ODBC DRIVER`。

```ini
[DM8 ODBC DRIVER]
Driver=/opt/dm8/drivers/odbc/libdodbc.so
```

以应用运行用户查询该条目。能查到驱动名和路径，说明注册对当前用户可见。实际能否加载驱动、登录数据库，还需运行客户端验证。

```sh
env -u ODBCSYSINI -u ODBCINSTINI odbcinst -q -d -n 'DM8 ODBC DRIVER'
```

项目自带的 `run-redconf-linux.sh` 会重新设置 `ODBCSYSINI` 和 `ODBCINSTINI`，所以原样运行它仍会读取应用自己的注册文件。使用系统级注册时，在部署环境中先准备好 Qt QODBC 插件、达梦配套库，以及这版驱动按名称查找的 `libodbcinst.so` 和 `libcrypto.so`，然后从安装目录直接启动客户端，使 unixODBC 使用默认配置路径。数据库地址仍填在 `config.json`，账号和密码仍由加密凭据文件提供。

```sh
env -u ODBCSYSINI -u ODBCINSTINI ./bin/RedConfClient
```

## 用连接串打开数据库

下面的片段假设调用方已经校验主机和端口，并从凭据文件读取 `user` 和 `password`。项目拒绝凭据中的分号、花括号和控制字符，避免拼接时改变 ODBC 参数的边界。

```cpp
const QRegularExpression delimiters(QStringLiteral(R"([;{}\x00-\x1f\x7f])"));
if (!QSqlDatabase::isDriverAvailable(QStringLiteral("QODBC"))
    || user.contains(delimiters) || password.contains(delimiters)) {
    return false;
}

auto db = QSqlDatabase::addDatabase(QStringLiteral("QODBC"), connectionName);
db.setDatabaseName(
    QStringLiteral("DRIVER={DM8 ODBC DRIVER};SERVER=%1;TCP_PORT=%2;UID=%3;PWD=%4")
        .arg(host, QString::number(port), user, password));

if (!db.open())
    return false;

QSqlQuery query(db);
return query.exec(QStringLiteral("SELECT 1 FROM DUAL")) && query.next();
```

`DRIVER` 对应当前 `odbcinst.ini` 中的驱动注册名。实测中，Qt 6.8 的 QODBC 会给 `open(user, password)` 传入的凭据加花括号，这版达梦驱动将花括号当作凭据内容。项目因此在完整连接串里设置 `UID` 和 `PWD`，调用无参 `open()`。连接串含有明文凭据，日志只记录连接或查询失败的阶段。

## 按加载顺序排查

1. **Qt 插件。** 在安装目录运行 `QT_DEBUG_PLUGINS=1 ./bin/run-redconf-linux.sh`，查看 Qt 是否找到并加载 `plugins/sqldrivers/libqsqlodbc.so`。用 `file` 核对插件和应用的架构，再用 `ldd plugins/sqldrivers/libqsqlodbc.so` 检查是否有 `not found`。
2. **达梦驱动。** 核对脚本生成的 `odbcinst.ini` 中的驱动名和实际路径。在与启动脚本相同的库搜索环境下，对 `bin/dameng-odbc/libdodbc.so` 执行 `ldd`，检查它的链接依赖。驱动在运行时按名称加载的库未必出现在 `ldd` 结果中，还要核对脚本生成的兼容链接。
3. **数据库连接。** 驱动加载成功后，再检查地址、端口和凭据。先用 `SELECT 1 FROM DUAL` 验证连接，再确认只读账号能执行应用需要的查询。

当时在麒麟上用干净构建验证了达梦直连查询和跨轮询连接复用，也从安装目录通过启动脚本完成了四屏启动验证。这几项分别覆盖数据库访问、连接生命周期，以及打包后的插件和动态库加载。
