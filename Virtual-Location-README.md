<div align="center">

# Virtual Location

**Android 系统级虚拟定位与轨迹模拟框架**

在系统层接管定位、GNSS、Wi-Fi、基站与蓝牙，把"人在哪里"变成可编程的参数。

[![License](https://img.shields.io/badge/license-GPL--3.0-blue.svg)](LICENSE)
[![Android](https://img.shields.io/badge/Android-8.0%2B-3DDC84?logo=android&logoColor=white)](#环境要求)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.2-7F52FF?logo=kotlin&logoColor=white)](#技术栈)
[![Compose](https://img.shields.io/badge/UI-Jetpack%20Compose-4285F4?logo=jetpackcompose&logoColor=white)](#技术栈)
[![LSPosed](https://img.shields.io/badge/libxposed-API%20100--102-6E56CF)](#环境要求)
[![ABI](https://img.shields.io/badge/ABI-arm64--v8a-orange)](#faq)

[功能特性](#功能特性) · [工作原理](#工作原理) · [兼容性](#兼容性) · [快速开始](#快速开始) · [从源码构建](#从源码构建) · [架构](#架构) · [FAQ](#faq) · [免责声明](#免责声明)

</div>

---

## 这是什么

**Virtual Location** 是一个基于 Xposed / LSPosed 的定位模拟框架，面向 Android 开发者与测试人员。

它不是通过系统「模拟位置」开关去骗一个 `Location` 对象，而是在**框架层**接管定位链路：

- 非全局模式：只在选定的目标应用进程内 Hook，改动范围可控、便于调试；
- 全局模式：注入 `system_server`、`com.android.phone`、`com.android.bluetooth`，让整机所有应用读到同一份"虚拟现实"。（编写中）

除了坐标本身，它还负责把「这串坐标看起来像真的」所需要的周边证据一起补齐：卫星星座、NMEA 语句、Wi-Fi 扫描列表、基站信息、BLE 广播、计步与加速度传感器时序。

> 适用场景：移动端定位功能测试、地理围栏与轨迹类应用的质量保障、位置相关风控策略研究、教学演示。
> 请勿用于考勤代打、位置欺诈或规避平台风控（见 [免责声明](#免责声明)）。

---

## 功能特性

### 定位注入

- **provider 全覆盖**：GPS / 网络 / 被动 / fused，`getLastLocation`、`requestLocationUpdates`、`PendingIntent` 回调、`LocationListener` 全链路接管
- **回调一致性**：时间戳、精度、速度、方位、海拔按物理模型生成，避免"静止却一直有速度"这类破绽
- **Google FusedLocationProvider** 与 **高德 / 百度 / 腾讯定位 SDK** 单独适配，SDK 内部缓存与回调均命中

### GNSS 仿真

- 多星座：GPS / 北斗 / GLONASS / Galileo，可配置卫星数量与星座分布
- 逐颗卫星的方位角、仰角、载噪比（C/N0）按物理模型漂移，而不是随机数
- **NMEA 0183** 语句生成（GGA / RMC / GSA / GSV / GNS 等），支持 talker 模式与自定义精度位数
- 首次定位时间（TTFF）、HDOP/PDOP/VDOP 可调，贴近真实冷启动表现

### 环境证据仿真

- **Wi-Fi**：扫描列表、已连接网络、信号强度、能力集、DHCP 信息、扫描时间抖动
- **基站**：LTE / NR 小区信息、MCC/MNC、RSRP/RSRQ/RSSI、邻区列表
- **蓝牙 / BLE**：扫描结果、广播包、iBeacon 载荷、发现与丢失时序

### 运动与轨迹

- **路线模拟**：地图选点、GPX 导入导出、途经点、停留时长、速度曲线，按路径真实插值
- **摇杆操控**：悬浮球方向控制，支持速度倍率与死区，适合"边走边调"
- **步态模型**：步频/步幅自适应速度，计步器与加速度传感器同步输出，支持步态录制后回放
- **海拔与噪声**：气压高度模型、GPS 精度抖动，避免"完美直线 + 零误差"的机器感

### 双注入方案

| 方案 | 注入目标 | 适用 | 特点 |
|---|---|---|---|
| `scoped` | 选定的目标应用进程 | 单应用调试 | 影响面小、可随时开关、无需重启系统服务 |
| `global` | `system_server` / phone / bluetooth | 全机生效 | 对未适配 SDK 的应用同样有效，需要重启系统服务或整机 |

### 多厂商适配

按 ROM 走不同的适配器，类名探测失败自动回落到 AOSP 基线：

`HyperOS / MIUI` · `ColorOS / OxygenOS` · `One UI` · `RedMagic (Nubia)` · `AOSP` · `Android 模拟器`

### 环境一致性与反检测

- `AppOps` 模拟定位标记、`Settings.Secure.mock_location`、开发者选项状态一并伪造
- Root / LSPosed 痕迹隐藏、进程与服务扫描过滤、系统属性覆盖
- 可选的原生层补丁（`libtttnative`），处理 Java 层够不到的检测点

### 数据采集与复用

- 采集真实环境的 Wi-Fi / 基站 / BLE 指纹，按位置 + 时间衰减复用，让虚拟坐标落在真实存在的网络环境里
- 可选接入 **WiGLE** 与 **OpenCellID**（使用你自己的 Token）反查周边热点与基站
- 全部数据保存在本地 Room 数据库，不上传任何服务器

### 扩展能力

- **脚本引擎**：Rhino，在 `location` / `step` / `gnss` 三个钩子点注入自定义逻辑，带类白名单与指令数上限
- **本地控制 API**：只监听 `127.0.0.1` 的 HTTP 接口，双 Token（完全控制 / 只读），自带限流与访问审计
- **广播与快捷磁贴**：自动化脚本、Tasker 类工具、通知栏磁贴均可控制
- **地图双引擎**：高德与百度可切换，支持离线地图与 WGS84 / GCJ-02 / BD-09 坐标转换

### 界面

- Jetpack Compose + Material 3，深色/浅色跟随系统
- 中文 / English 双语，可跟随系统语言
- 新手模式与极客面板分离：常用参数一键切换，底层参数按需展开

---

## 工作原理

```text
                     ┌──────────────────────────────┐
                     │  目标应用（任意进程）          │
                     │  LocationManager / SDK 回调   │
                     └──────────────┬───────────────┘
                                    │ Binder
                     ┌──────────────▼───────────────┐
   scoped 模式 ─────►│  应用进程内 Hook              │
                     │  AMap / Baidu / Tencent /     │
                     │  Google Fused / 系统 API      │
                     └──────────────┬───────────────┘
                                    │
   global 模式 ─────►┌──────────────▼───────────────┐
                     │  system_server                │
                     │  LocationManagerService       │
                     │  WifiService / Telephony /    │
                     │  Bluetooth / AppOps           │
                     └──────────────┬───────────────┘
                                    │ 共享状态
                     ┌──────────────▼───────────────┐
                     │  Virtual Location 主程序      │
                     │  地图 / 路线 / 参数 / 控制面   │
                     └──────────────────────────────┘
```

主程序负责"决定去哪、怎么去"，Hook 层负责"让系统与目标应用相信"。两者通过配置通道同步，修改参数无需重新注入。

---

## 兼容性

### 系统

| 系统 | 适配器 | 状态 |
|---|---|---|
| 小米 HyperOS 4 / HyperOS 3 | `HyperOsVendor` | ✅ 定位注入实测可用 |
| OPPO ColorOS 16 / Android 16 | `ColorOs16Vendor` | ⚠️ 部分组件实测通过 |
| 三星 One UI | `OneUiVendor` | ❔ 未验证 |
| 红魔 RedMagic OS | `NubiaVendor` | ⚠️ 类名解析与注入成功，业务侧待复验 |
| 原生 AOSP / 类原生 | `AospVendor` | ❔ 基线适配器 |
| Android 模拟器 | `EmulatorVendor` | ❔ 代码就绪，实机待补 |

> 状态含义：✅ 实测通过 · ⚠️ 部分可用 · ❔ 未验证。
> 适配进展与实测记录维护在 `docs/adaptation/`（目录编写中）。

### 运行环境

| 项 | 要求 |
|---|---|
| Android | 8.0（API 26）及以上 |
| 架构 | `arm64-v8a` |
| Root | Magisk / KernelSU / APatch 任一 |
| 框架 | LSPosed（libxposed API 100–102） |

---

## 快速开始

1. **准备环境**：确认设备已 Root，并已安装可用的 LSPosed。
2. **安装**：从 [Releases](https://github.com/lichongg/Virtual-Location/releases) 下载 APK 并安装。
3. **启用模块**：在 LSPosed 管理器中启用 ttt。
4. **勾选作用域**
   - 非全局模式：勾选你要模拟的目标应用；
   - 全局模式：勾选「系统框架」（`system_server`），并按需勾选电话、蓝牙。（待上线）
5. **重启生效**：非全局模式重启目标应用即可；全局模式建议重启手机。
6. **开始模拟**：打开 App → 地图选点或导入路线 → 设置速度/步态 → 启动。（建议使用摇杆模式方便控制）

> 建议先在「诊断」页确认各系统组件是否全部命中，再开始调参。

---

## 从源码构建

### 环境

- JDK 21
- Android SDK Platform 37 + Build Tools
- Android NDK（构建原生层时需要）
- Gradle（使用仓库自带 Wrapper）

### 命令

```bash
git clone https://github.com/lichongg/Virtual-Location.git
cd Virtual-Location

# 调试包
./gradlew :app:assembleDebug

# 发布包（需先配置 signingConfigs）
./gradlew :app:assembleRelease
```

### 签名

发布包请使用**自己的 release keystore**，并开启 v2 + v3 签名：

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file(System.getenv("VL_KEYSTORE") ?: "release.jks")
            storePassword = System.getenv("VL_STORE_PASSWORD")
            keyAlias = System.getenv("VL_KEY_ALIAS")
            keyPassword = System.getenv("VL_KEY_PASSWORD")
        }
    }
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}
```

> 当前版本仍在用调试证书签名，正式发布前会切换到独立 release 密钥（见 [路线图](#路线图)）。

---

## 架构

### 模块

| 模块 | 职责 |
|---|---|
| `app` | Compose UI、地图、导航、设置、新手引导 |
| `core-geo` | 坐标转换、GNSS/NMEA 模型、路线与步态引擎、厂商识别 |
| `core-data` | Room 数据层、Settings/配置、采集与复用策略 |
| `service` | 前台服务、悬浮摇杆、采集服务、控制面（HTTP / 广播 / 磁贴） |
| `xposed` | 注入入口与全部 Hook 实现、诊断与状态上报、原生桥 |

### 关键组件

```text
xposed/
├─ LocationHooker              模块入口（java_init.list 指定）
├─ hooks/
│  ├─ SystemLocationService    system_server 定位服务
│  ├─ SystemWifiService        Wi-Fi 扫描与连接
│  ├─ SystemTelephonyService   基站信息
│  ├─ SystemBluetoothService   BLE 扫描与广播
│  ├─ GnssStatus / Nmea        GNSS 状态与语句
│  ├─ SensorStep               计步与加速度
│  ├─ AMap / Baidu / Tencent   第三方定位 SDK
│  └─ vendor/                  各 ROM 适配器与进程策略
└─ diagnostics/                Hook 状态、守卫报告、热重载
```

控制面：`ControlPlaneSupervisor` → `ControlGate`（能力分级 + 恒时比较）→ `ControlExecutor`，失败计数触发限流，所有访问写入审计日志。

### 技术栈

| 层 | 选型 |
|---|---|
| 语言 | Kotlin 2.2 |
| UI | Jetpack Compose · Material 3 · Navigation |
| DI | Koin |
| 存储 | Room（Wi-Fi / 基站 / BLE / 轨迹）· DataStore 规划中 |
| 序列化 | kotlinx.serialization |
| 网络 | OkHttp |
| 脚本 | Mozilla Rhino |
| 地图 | 高德地图 SDK · 百度地图 SDK |
| 注入 | libxposed API |
| 原生 | C++ / JNI（`libtttnative`） |

---

## 本地控制 API

默认**关闭**。在「外部控制」页开启后，仅监听回环地址，默认端口 `8765`。

```bash
# 查询状态（只读 Token 即可）
curl -H "X-TTT-Token: <readonly-token>" http://127.0.0.1:8765/status

# 下发指令（需要完全控制 Token）
curl -X POST http://127.0.0.1:8765/command \
  -H "X-TTT-Token: <full-token>" \
  -H "Content-Type: application/json" \
  -d '{"type":"start","lat":39.9087,"lng":116.3975}'
```

- 双 Token：`full` 可读可写，`readonly` 仅 `/status`
- 恒时比较校验，60 秒内 10 次失败进入冷却
- 请求头 ≤ 8 KB、请求体 ≤ 64 KB、5 秒超时
- 也可通过广播（`com.ttt.t.START` / `STOP` / `PAUSE` / `RESUME` / `SET_LOCATION` / `SET_SPEED` / `SET_TOGGLE` / `STATUS`）或通知栏磁贴控制

---

## 路线图

- [ ] 切换到独立 release 签名，启用 v3 / v4
- [ ] 开启 R8 混淆与资源裁剪，目标体积下降 25%+
- [ ] 地图引擎可选安装，默认只保留一个以缩减包体
- [ ] 配置模型重构为版本化 `AppConfig`，支持平滑迁移
- [ ] 厂商适配器插件化，社区可独立贡献新 ROM
- [ ] 完善自动化测试：控制面、坐标转换、NMEA、路线插值
- [ ] 多语言扩展与系统级「按应用设置语言」

---

## FAQ

**需要 Root 吗？**
需要。没有 Root 无法把代码注入 `system_server` 或其他应用进程，定位模拟就只能停留在系统「模拟位置」的层面，很容易被识别。

**为什么只有 arm64-v8a？**
目前只维护 arm64 构建。需要 x86_64 / armeabi-v7a 的话可以自行从源码编译，或在 Issue 里说明场景。

**开启了但目标应用还是显示旧位置？**
按顺序排查：作用域是否勾选 → 目标应用是否已重启 → 诊断页各组件是否 `found` → 目标应用是否使用了未被 Hook 的私有通道（例如自带 C 层定位）。

**会被检测出来吗？**
项目在坐标、GNSS、网络环境、传感器多个维度做了对齐，但检测与反检测是持续对抗，**不承诺对任何应用都不可见**。

**会上传我的位置数据吗？**
不会。应用没有自建后端，所有采集数据保存在本地数据库；只有你主动配置 WiGLE / OpenCellID Token 并触发查询时，才会向对应服务发送坐标区间。

**支持哪些语言？**
中文与 English，可跟随系统。欢迎通过 PR 补充翻译。

---

## 参与贡献

- 提交 Issue 时请附上：系统版本、机型、Root 方案、LSPosed 版本、诊断页截图
- 新增 ROM 适配请放在 `xposed/.../hooks/vendor/profiles/` 下，并附实测记录
- 代码风格：Kotlin 官方风格，提交前跑 `./gradlew spotlessCheck detekt`

---

## 免责声明

本项目仅供**学习研究、自有设备测试与授权范围内的安全评估**使用。

- 使用者应自行确保其行为符合所在地法律法规、设备厂商条款以及目标应用的服务协议；
- **禁止**将本项目用于考勤代打、位置欺诈、骗取补贴或规避平台风控等用途；
- 因使用本项目产生的一切后果由使用者自行承担，作者不对任何直接或间接损失负责；
- 本项目按 GPL-3.0 提供，**不附带任何担保**。

如果你不确定自己的使用场景是否合规，请不要使用。

---

## 许可证

[GPL-3.0](LICENSE)

## 致谢

- [libxposed](https://github.com/libxposed) —— 现代 Xposed API
- [LSPosed](https://github.com/LSPosed/LSPosed) —— 运行框架
- [高德地图 SDK](https://lbs.amap.com/) / [百度地图 SDK](https://lbsyun.baidu.com/) —— 地图与定位能力
- [Mozilla Rhino](https://github.com/mozilla/rhino) —— 脚本引擎

<div align="center">

如果这个项目对你有帮助，欢迎点一个 ⭐

</div>
