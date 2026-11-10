# 银河麒麟 Qt 项目接入 OpenSSL：用配套头文件和系统 libcrypto 构建 AES

签到客户端把数据库账号和密码保存在加密文件里。凭据工具在终端读取账号和密码并写入文件，真实签到源在建立数据库连接前读取并解密。两个入口共用 `CheckInCredentials.cpp`，所以加密依赖由凭据模块统一声明。

本文以签到客户端提交 `da0cb20` 为例。历史实测环境为银河麒麟 V10 SP3 x86_64、Qt 6.8.3 和 OpenSSL 1.1.1f，同次验证使用达梦 DM8 的 2026-07-10 银河麒麟 V10 SP3 x86_64 客户端包。

## AES-256-GCM 用到 OpenSSL 的哪个库

`<openssl/evp.h>` 提供 EVP 加解密接口，项目通过 `EVP_aes_256_gcm()` 选择 AES-256-GCM。`<openssl/rand.h>` 提供 `RAND_bytes()`，每次加密时生成新的 12 字节 nonce。文件依次保存版本字节、nonce、密文和 16 字节认证标签。解密时先设置标签，等 `EVP_DecryptFinal_ex()` 校验通过后再解析明文。项目的密钥内置在程序中，凭据文件仍按敏感文件管理。[OpenSSL EVP 文档](https://docs.openssl.org/1.1.1/man3/EVP_EncryptInit/) · [OpenSSL 随机数接口文档](https://docs.openssl.org/1.1.1/man3/RAND_bytes/)

这些函数由 OpenSSL 的加密库 `libcrypto` 提供。凭据模块只需要 Crypto 组件，`libssl` 提供的 TLS 接口没有参与这段代码。

## 为什么系统有 OpenSSL，CMake 还是找不到

在目标麒麟机器上分别查看命令行工具、软件包状态和动态库：

```sh
openssl version
dpkg-query -W -f='${Package}\t${Status}\t${Version}\n' \
    openssl libssl1.1 libssl-dev
ldconfig -p | grep -F libcrypto.so.1.1
```

| 检查对象 | 在这台麒麟机器上的作用 |
| --- | --- |
| `openssl` | 命令行工具，可查看版本。命令能运行只说明命令行工具的运行依赖可用。 |
| `libssl1.1` | 运行库包，提供 `libcrypto.so.1.1`。本次解析到 `/usr/lib/x86_64-linux-gnu/libcrypto.so.1.1`。 |
| `libssl-dev` | 开发包，提供 EVP、随机数等头文件，以及构建工具通常查找的未版本化 `libcrypto.so` 链接名。 |

目标机已有 1.1.1f 运行库，缺少开发包。编译器需要头文件中的类型和函数声明，链接器需要定位实现这些函数的动态库。这里可以把这两项分别解决：头文件放进项目，CMake 直接查找系统已有的带版本号的库。

## 从匹配的麒麟软件包提取头文件

先确认目标机 `libssl1.1` 的版本和架构，再准备同版本、同架构的麒麟 `libssl-dev` 包。本次使用 `libssl-dev_1.1.1f-1kylin2.23k0.7_amd64.deb`。从项目根目录执行以下命令即可提取头文件，`dpkg-deb -x` 只解包。

```sh
package=./libssl-dev_1.1.1f-1kylin2.23k0.7_amd64.deb
headers=vendor/openssl/include/linux-x86_64

(
    set -eu
    unpacked=$(mktemp -d)
    trap 'rm -rf "$unpacked"' EXIT
    dpkg-deb -x "$package" "$unpacked"
    mkdir -p "$headers/openssl"
    cp -R "$unpacked/usr/include/openssl/." "$headers/openssl/"
    cp "$unpacked/usr/include/x86_64-linux-gnu/openssl/opensslconf.h" \
        "$headers/openssl/"
)
```

`opensslconf.h` 来自同一软件包的架构专用目录。把它与其余头文件放到同一套 `openssl/` 目录后，编译器就能通过一个包含路径解析所有头文件。项目在 `vendor/openssl` 里保留软件包来源、版本、校验值和许可。系统运行库仍由目标机的软件包提供。

## 在 CMake 里链接系统的 libcrypto.so.1.1

下面是项目按平台选择 Crypto 依赖的核心片段。`RedConfCheckInCredentials` 是包含凭据实现的静态库目标，客户端的真实签到源和凭据工具分别使用它。

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux"
   AND CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
    find_library(REDCONF_SYSTEM_LIBCRYPTO NAMES libcrypto.so.1.1)
    if(NOT REDCONF_SYSTEM_LIBCRYPTO)
        message(FATAL_ERROR "缺少系统 libcrypto.so.1.1，请安装匹配的运行库。")
    endif()

    add_library(RedConfSystemCrypto SHARED IMPORTED)
    set_target_properties(RedConfSystemCrypto PROPERTIES
        IMPORTED_LOCATION "${REDCONF_SYSTEM_LIBCRYPTO}"
        INTERFACE_INCLUDE_DIRECTORIES
            "${PROJECT_SOURCE_DIR}/vendor/openssl/include/linux-x86_64")
    set(redconf_crypto_target RedConfSystemCrypto)
else()
    find_package(OpenSSL REQUIRED COMPONENTS Crypto)
    set(redconf_crypto_target OpenSSL::Crypto)
endif()

target_link_libraries(RedConfCheckInCredentials
    PUBLIC Qt6::Core
    PRIVATE ${redconf_crypto_target})
```

Linux x86_64 构建时，导入目标把仓库里的包含目录传给编译器，把系统库的路径交给链接器。显式查找 `libcrypto.so.1.1` 后，构建机无需为这个项目安装 `libssl-dev` 或手工创建 `libcrypto.so`。macOS 上的凭据工具通过 `OpenSSL::Crypto` 使用本机 OpenSSL。

## 从干净构建验证到安装目录

在签到客户端仓库根目录，将 `QT_ROOT` 指向麒麟上的 Qt Kit，使用一个尚未配置过的构建目录。发布规则还需要匹配的 Fcitx 插件和 `patchelf`。[《Qt 6 Linux 打包实战》](Qt6%20Linux打包实战与Fcitx插件分发.md)

```sh
cmake --preset release -B build/crypto-check -DREDCONF_PACKAGE_LINUX=ON
cmake --build build/crypto-check --target RedConfClient RedConfCheckInCredentialsTool
cmake --install build/crypto-check --prefix "$PWD/build/crypto-install" \
    --component RedConfRuntime

ldd build/crypto-install/bin/RedConfClient | grep -F libcrypto
ldd build/crypto-install/bin/RedConfCheckInCredentials | grep -F libcrypto
find build/crypto-install -name 'libcrypto.so*' -print
```

历史验证中，客户端和凭据工具都解析到系统的 `/usr/lib/x86_64-linux-gnu/libcrypto.so.1.1`。安装目录里没有这份动态库。项目的 Qt 部署脚本显式排除了 `libcrypto.so.1.1`，因此目标机仍需保留对应的运行库。

达梦驱动还有一条独立的运行时依赖。它会按 `libcrypto.so` 这个未版本化名称查找加密库，Linux 启动脚本因此在用户运行目录创建指向系统 `libcrypto.so.1.1` 的兼容软链接，并让应用进程能搜索到它。这个链接在启动时生成，凭据模块的 CMake 链接直接使用带版本号的系统库。[《银河麒麟 Qt 6 连接达梦 DM8》](银河麒麟Qt6连接达梦DM8.md)
