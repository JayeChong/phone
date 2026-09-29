# 手持终端分阶段实施计划

计划日期：2026-09-30

状态：设计已批准，等待执行

## 目标

从当前只有文档的仓库开始，建立可构建、可测试、可在目标开发板上验证的 ESP-IDF 固件，并按以下顺序交付：

1. 硬件冒烟与关键技术验证。
2. 桌面内核与 APP 生命周期。
3. 电话服务与电话 APP。
4. 相机、相册与资源预算。
5. GNSS、阶段 0 已验证的数据链路、HTTPS 与单家庭云端协议。
6. 整机稳定性和发布基线。

本计划依据：

- [手持终端需求文档](../../需求.md)
- [桌面与 APP 架构设计](../specs/2026-09-29-handheld-desktop-architecture-design.md)
- [需求与架构评审处置台账](../specs/2026-09-29-requirements-architecture-review.md)

## 范围边界

本计划详细覆盖 4G 手持终端固件。以下内容不与固件主线混在一次实现中：

- 云端后端和家属地图前端：本计划先完成 API 契约、设备端 HTTPS 客户端和本地 Mock 验证；后端技术栈确认后另写独立设计与实施计划。
- 1.75C 圆屏：等待设备 API 和云端 API 稳定后另写计划。
- 外壳、结构件和量产 PCB：等待最终接线、功耗和热量数据后另写计划。

## 执行原则

- 阶段 0 是硬门槛。共享 I2C、启动稳定性、移动卡语音预验证或 USB Host 候选数据链路任一失败时，先修正硬件/架构，不继续依赖该事实的功能。
- 每个任务先建立可失败的测试、探针或验收记录，再实现功能。
- LVGL 只能由 UI Task 调用；UART 只能由 `Modem Manager` 调用。
- 固定 Worker 和有界队列优先，不按请求动态创建任务。
- 每个阶段结束时保存构建日志、关键性能数据和真机验证结果。
- 计划中的提交检查点只表示建议的变更边界，执行时未经用户要求不自动提交。

## 目标工程结构

```text
firmware/
├── CMakeLists.txt
├── dependencies.lock
├── idf_component.yml
├── partitions.csv
├── sdkconfig.defaults
├── main/
│   ├── CMakeLists.txt
│   └── app_main.c
├── components/
│   ├── board_support/
│   ├── event_bus/
│   ├── app_runtime/
│   ├── ui_port/
│   ├── desktop_shell/
│   ├── settings_store/
│   ├── diagnostics/
│   ├── battery_monitor/
│   ├── modem_manager/
│   ├── call_service/
│   ├── network_link/
│   ├── location_service/
│   ├── media_worker/
│   ├── network_worker/
│   ├── phone_app/
│   ├── camera_app/
│   ├── gallery_app/
│   └── settings_app/
└── test_apps/
    ├── unit/
    │   ├── CMakeLists.txt
    │   └── main/
    ├── hardware_smoke/
    ├── modem_smoke/
    └── usb_host_probe/
docs/
├── api/
│   └── device-cloud-api.md
└── verification/
    ├── phase-0-hardware-smoke.md
    ├── phase-1-desktop.md
    ├── phase-2-call.md
    ├── phase-3-media-memory.md
    ├── phase-4-location-network.md
    └── phase-5-stability.md
```

每个组件只公开自身 `include/` 下的头文件。组件内部 LVGL 对象、UART 句柄、文件句柄和可变状态不得跨边界暴露。

### 测试与探针约定

- `test_apps/unit` 是独立 ESP-IDF Unity 测试工程，通过 `EXTRA_COMPONENT_DIRS` 引入固件组件及其测试代码；测试完成必须输出 Unity 汇总和失败数，打开 monitor 不等于测试通过。
- `hardware_smoke`、`modem_smoke` 和 `usb_host_probe` 是三个独立 ESP-IDF probe 工程，共用正式组件，不与正式 `app_main.c` 编译到同一个目标。
- 纯解析器、状态机、队列和 codec 优先使用 Unity 自动测试；依赖目标芯片的测试由 `pytest-embedded` 烧录、抓取串口并断言结果。
- 真机人工操作项使用 `docs/verification/` 检查表记录，不伪装成自动化测试。

## 阶段 0：工程与硬件冒烟

### 任务 0.1：安装并锁定构建环境

**修改文件**

- 更新 `.gitignore`
- 新建 `firmware/CMakeLists.txt`
- 新建 `firmware/main/CMakeLists.txt`
- 新建 `firmware/main/app_main.c`
- 新建 `firmware/sdkconfig.defaults`
- 新建 `firmware/partitions.csv`
- 新建 `firmware/idf_component.yml`
- 新建 `firmware/test_apps/unit/CMakeLists.txt`
- 新建 `firmware/test_apps/unit/main/CMakeLists.txt`
- 新建 `firmware/test_apps/unit/main/test_main.c`
- 新建 `firmware/test_apps/unit/pytest_unit.py`
- 新建 `firmware/test_apps/hardware_smoke/` 工程骨架
- 新建 `firmware/test_apps/modem_smoke/` 工程骨架
- 新建 `firmware/test_apps/usb_host_probe/` 工程骨架

**步骤**

1. 安装 ESP-IDF v5.5.5 及其匹配的 CMake、Ninja、Python 环境。
2. 创建最小 ESP32-S3 工程，`app_main` 只打印固件版本、芯片信息、Flash、内部 SRAM 和 PSRAM 信息。
3. `sdkconfig.defaults` 启用 ESP32-S3、8MB PSRAM、16MB Flash、任务栈溢出检查和堆完整性检查。
4. 初始 16MB 分区表使用以下完整布局；应用分区均按 64KB 对齐，数据分区按 4KB 对齐：

    | 名称 | 类型/子类型 | Offset | Size |
    | :--- | :--- | :--- | :--- |
    | `nvs` | data/nvs | `0x9000` | `0x6000` |
    | `otadata` | data/ota | `0xF000` | `0x2000` |
    | `phy_init` | data/phy | `0x11000` | `0x1000` |
    | `nvs_keys` | data/nvs_keys | `0x12000` | `0x1000` |
    | `coredump` | data/coredump | `0x13000` | `0x10000` |
    | `factory` | app/factory | `0x30000` | `0x400000` |
    | `ota_0` | app/ota_0 | `0x430000` | `0x400000` |
    | `ota_1` | app/ota_1 | `0x830000` | `0x400000` |
    | `storage` | data/fat | `0xC30000` | `0x3D0000` |

    首次链接后验证应用不超过 4MB；若超出，在任何设备写入用户数据前重新评审分区，不能让不同设备使用不兼容布局。
5. `.gitignore` 忽略 `firmware/build/`、`firmware/sdkconfig`、`firmware/sdkconfig.old`、崩溃转储和本地证书/凭据。
6. 为三个 probe 工程定义各自唯一入口；正式固件和 probe 不通过条件编译共享多个 `app_main`。
7. `test_apps/unit` 的 CMake 通过 `EXTRA_COMPONENT_DIRS` 引入组件，组件 `test/CMakeLists.txt` 用 `idf_component_register` 注册测试源；`pytest_unit.py` 使用 `pytest-embedded` 烧录、等待 Unity 汇总并在失败数非零时返回失败。
8. 启用 bootloader app rollback。新 OTA 镜像首次启动后，只有 NVS 可读、UI Task 启动、堆完整性检查通过并稳定运行 30 秒，才调用 `esp_ota_mark_app_valid_cancel_rollback()`；不要求 SIM、GNSS 或网络在线。标记前发生 panic、看门狗或重复复位时由 bootloader 回滚，factory 分区作为有线恢复基线。
9. 增加 OTA 回滚探针：一版故意不标记有效，确认重启后回滚；一版通过启动健康检查，确认标记有效后不回滚。

**验证**

```bash
cd firmware
idf.py set-target esp32s3
idf.py reconfigure
idf.py build
```

通过条件：从干净工作区可重复构建，版本输出明确为 ESP-IDF v5.5.5，链接产物未超出分区。

### 任务 0.2：锁定组件与板级配置

**新增文件**

- `firmware/components/board_support/CMakeLists.txt`
- `firmware/components/board_support/include/board_pins.h`
- `firmware/components/board_support/include/board_config.h`
- `firmware/components/board_support/board_support.c`
- `firmware/components/board_support/test/test_board_config.c`

**步骤**

1. 在组件管理器中锁定 LVGL v9.6.0、`esp32-camera` v2.1.8 和明确版本的 `esp_lvgl_port`。
2. 将需求中的最终 GPIO 放入唯一板级头文件，其他组件不得重复写数字引脚。
3. 对编译期冲突做静态断言：显示 SPI、TF、UART、USB、摄像头数据脚不得重复；GPIO0 禁止分配。
4. 明确 GPIO15/16 是共享 I2C/SCCB 总线，GPIO40 只属于 A7670 RI。
5. 生成并提交 `dependencies.lock`。
6. `esp32-camera` v2.1.8 的 SCCB 复用的是 legacy `driver/i2c.h` 端口编号，不是新 I2C master bus handle；阶段 0 统一使用兼容的 legacy I2C driver 管理共享端口，触摸也通过同一端口访问。除非单独适配相机 SCCB 层，否则不得混用新旧 I2C 驱动。

**测试**

- 板级配置测试验证关键引脚、分辨率、Flash/PSRAM 预期值和共享总线标记。
- 故意制造重复引脚时应编译失败，恢复后重新构建通过。

### 任务 0.3：建立结构化日志和验证记录

**新增文件**

- `firmware/components/diagnostics/include/diagnostics.h`
- `firmware/components/diagnostics/diagnostics.c`
- `docs/verification/phase-0-hardware-smoke.md`

**步骤**

1. 定义启动原因、内部 SRAM/PSRAM、最大连续块、任务栈水位和复位原因的统一日志格式。
2. 每次硬件试验记录固件 commit、开发板版本、拨码状态、SIM、供电方式和结果。
3. 记录表先填写“未执行”，不得用推测结果代替真机结果。

**验证**

- 启动日志可清楚区分上电、软件复位、看门狗和异常复位。
- 验证文档能追溯到唯一固件构建。

### 任务 0.4：验证共享 I2C/SCCB

**新增文件**

- `firmware/components/board_support/include/shared_i2c.h`
- `firmware/components/board_support/shared_i2c.c`
- `firmware/components/board_support/test/test_shared_i2c.c`
- `firmware/test_apps/hardware_smoke/main/hardware_smoke.c`

**步骤**

1. 由 `board_support` 使用 legacy `driver/i2c.h` 创建 GPIO15/16 的唯一 I2C master 端口，所有访问由同一个总线互斥保护。
2. 初始化 FT6336U 并确认触摸地址可探测。
3. 配置 `esp32-camera`：`pin_sccb_sda = -1`、`pin_sccb_scl = -1`，通过 `sccb_i2c_port` 复用已存在总线。
4. 初始化 OV5640，读取传感器 ID，连续采集 JPEG 帧。
5. 在采集期间持续读取触摸，验证总线不会死锁、超时或被重复卸载。
6. 关闭相机采集时只让传感器软件休眠，不关闭 CAM 供电。
7. 初始化和反初始化测试必须证明相机不会删除自己不拥有的共享 I2C driver，触摸销毁也不能卸载仍被相机使用的端口。

**真机验证**

- 连续 10 分钟触摸采样和每秒一次 JPEG 采集。
- 统计 I2C 超时、NACK、相机失败和触摸丢样次数。
- 任一总线死锁或驱动互相卸载都视为失败，先修正再继续。

### 任务 0.5：点亮显示并建立性能基线

**新增文件**

- `firmware/components/ui_port/CMakeLists.txt`
- `firmware/components/ui_port/include/ui_port.h`
- `firmware/components/ui_port/display_port.c`
- `firmware/components/ui_port/touch_port.c`
- `firmware/components/ui_port/ui_task.c`

**步骤**

1. 初始化 ST7796S、SPI DMA、LVGL v9.6.0 和 FT6336U 输入设备。
2. `UI Task` 固定到 Core 1，并成为唯一允许调用 LVGL 的任务。
3. 先使用 40MHz SPI，记录全屏纯色、局部矩形和触摸反馈性能。
4. 依次验证 60MHz、80MHz；每档运行 30 分钟并检查错行、闪屏、撕裂和 SPI 错误。
5. 选择最高稳定频率写入 `board_config.h`，不以单次成功作为定版依据。

**通过条件**

- 触摸到首个视觉反馈不高于 50ms。
- 局部更新目标 30 FPS。
- 全屏滑动至少 15 FPS；80MHz 稳定时目标至少 20 FPS。

### 任务 0.6：建立 Modem 原始链路冒烟程序

**新增文件**

- `firmware/components/modem_manager/include/modem_transport.h`
- `firmware/components/modem_manager/uart_transport.c`
- `firmware/test_apps/modem_smoke/main/modem_smoke.c`

**步骤**

1. 初始化 GPIO17/18 UART 115200，并建立独立 RX 任务和有界字节环形缓冲。
2. 仅在冒烟程序中顺序执行 `AT`、SIM 状态、网络注册、信号、模组版本和 GNSS 查询。
3. 打印原始行并标记命令响应或 URC，暂不实现完整调度器。
4. 使用临时中国移动卡完成真实呼出、呼入、接听、主动挂断和对方挂断。
5. 记录通话期间 GNSS 查询及数据链路表现，保存实际 URC 样本。

**通过条件**

- UART 无乱码和持续溢出。
- 五类真实通话操作均有可解释的 AT/URC 记录。
- 临时卡结果只关闭预验证项，不关闭最终 SIM 上线门槛。

### 任务 0.7：探测并验证 A7670 USB Host 数据链路

**新增文件**

- `firmware/components/network_link/CMakeLists.txt`
- `firmware/components/network_link/include/network_link.h`
- `firmware/components/network_link/network_link.c`
- `firmware/components/network_link/test/test_network_link_state.c`
- `firmware/test_apps/usb_host_probe/main/usb_host_probe.c`

**步骤**

1. 在 USB 拨码为 OFF 的正常运行候选状态下，使用 ESP-IDF USB Host Library 枚举 A7670，保存 VID/PID、configuration、interface、class/subclass/protocol 和 endpoint descriptors。
2. 根据实际 descriptors 确认是否存在可由 ESP-IDF USB Host CDC-ACM 驱动访问的数据接口；不得把 TinyUSB device 集成当作 Host CDC 方案。
3. 如果 CDC-ACM 成立，实现 USB 字节流到 PPP/netif 的最小适配，获取 IP、DNS 和默认路由；将实际拨号/会话流程写入验证记录。
4. 通过互联网可达的 HTTPS 测试端点执行小 JSON POST 和 200KB 文件 POST。
5. 数据传输期间持续接收 UART URC，并验证来电事件可被及时观察。
6. 重复断开、重连、模组重启和弱网恢复。
7. 如果没有可用 CDC 数据接口，或 PPP/netif 适配无法稳定工作，停止依赖上传链路的后续任务并重新评审高速 UART 或模组其他接口；不允许静默回退到 115200 UART。

**通过条件**

- 保存并人工复核 A7670 USB descriptors。
- 选定链路连续完成 20 次建链/断链循环。
- 200KB 上传不经过 115200 UART，UART URC 不丢失。
- 链路失败有确定超时和可恢复状态，不无限阻塞。

### 任务 0.8：执行最终接线启动稳定性矩阵

**修改文件**

- `docs/verification/phase-0-hardware-smoke.md`

**步骤**

1. 记录最终屏幕、触摸、OV5640、TF、4G、天线和供电接线，并拍照留档。
2. 在正常运行拨码状态下完成 50 次完全断电冷启动，每次确认进入应用、触摸可用、相机可探测和 Modem 可响应。
3. 完成 50 次 RESET 热重启，确认 LCD/FT6336U 软件复位后可恢复。
4. 验证 BOOT + RESET 进入下载模式、退出下载模式和重新烧录。
5. 覆盖 CAM ON/OFF、4G ON、USB 正常运行/电脑调试状态及 USB 供电/电池供电组合；不测试可能损坏硬件的非法组合。
6. 任一失败记录拨码、供电、复位原因和串口日志；修正后重新执行完整矩阵，不只重试失败一次。

### 阶段 0 门槛

只有以下项目全部通过，才进入桌面内核和完整业务实现：

- GPIO15/16 共享总线稳定。
- 50 次冷启动、50 次 RESET 和下载模式验证通过。
- 临时移动卡语音预验证通过。
- A7670 USB descriptors 已确认，选定的数据链路可稳定建链并与 UART 并行；若 USB 候选路径失败，重新评审后的替代链路已通过同等验证。
- 40MHz 显示与触摸基线通过。
- 任务 0.8 的完整启动矩阵通过并留有记录。

## 阶段 1：桌面内核

### 任务 1.1：定义跨组件事件契约

**新增文件**

- `firmware/components/event_bus/include/system_event.h`
- `firmware/components/event_bus/include/system_command.h`
- `firmware/components/event_bus/include/event_bus.h`
- `firmware/components/event_bus/event_bus.c`
- `firmware/components/event_bus/test/test_event_bus.c`

**步骤**

1. 事件头包含类型、来源、会话 ID、代次、负载长度和资源释放策略。
2. 建立普通 UI 队列和高优先级系统队列；来电、挂断和关键故障只走高优先级队列。
3. 对位置、电量和进度使用“只保留最新值”的合并槽，不堆积历史事件。
4. 测试队列满、重复事件、旧会话事件和资源释放。

**通过条件**

- 旧会话事件不能更新新页面。
- 高优先级事件不被普通状态洪水挤掉。

### 任务 1.2：实现 APP 注册和生命周期

**新增文件**

- `firmware/components/app_runtime/include/app_descriptor.h`
- `firmware/components/app_runtime/include/app_runtime.h`
- `firmware/components/app_runtime/app_registry.c`
- `firmware/components/app_runtime/app_runtime.c`
- `firmware/components/app_runtime/test/test_app_lifecycle.c`

**步骤**

1. 实现静态 `AppRegistry` 和唯一 APP ID 校验。
2. 实现 `Closed`、`Foreground`、`BackgroundCached`、`UiReclaimed` 状态。
3. 第一版后台页面缓存上限设为一个，最近任务上限设为四个。
4. 实现递增会话 ID、轻量状态保存和 UI 重建。
5. 使用假 APP 测试打开、切换、回收、恢复和关闭。

### 任务 1.3：实现桌面、页面栈和手势仲裁

**新增文件**

- `firmware/components/desktop_shell/include/desktop_shell.h`
- `firmware/components/desktop_shell/desktop_shell.c`
- `firmware/components/desktop_shell/page_manager.c`
- `firmware/components/desktop_shell/gesture_router.c`
- `firmware/components/desktop_shell/system_overlay.c`
- `firmware/components/desktop_shell/test/test_gesture_router.c`

**步骤**

1. 实现固定网格、自动分页和八个测试图标。
2. 实现 APP 独立页面栈及系统覆盖层恢复点。
3. 实现底部 24px 起始区、72px 上划阈值、12px 方向锁和任务卡关闭阈值。
4. 手势判断提取为不依赖 LVGL 对象的纯状态机并覆盖边界测试。
5. 真机连续切换桌面页 100 次，记录误触和堆变化。

### 任务 1.4：实现最近任务快照

**新增文件**

- `firmware/components/desktop_shell/task_manager.c`
- `firmware/components/media_worker/include/media_job.h`
- `firmware/components/media_worker/media_worker.c`
- `firmware/components/media_worker/test/test_media_queue.c`

**步骤**

1. APP 隐藏前由 UI Task 使用 LVGL Snapshot 捕获 APP 根对象。
2. 将图像和会话 ID 交给固定 `Media Worker`，缩放为 160×240 RGB565。
3. 结果返回 UI 队列；旧会话、已关闭 APP 或任务卡已删除时释放图像。
4. 注入分配失败、队列满和缩放超时，验证退化为图标卡。
5. 禁止使用 LCD 显存读回。

### 任务 1.5：实现设置与资源监控 APP

**新增文件**

- `firmware/components/settings_store/include/settings_store.h`
- `firmware/components/settings_store/settings_store.c`
- `firmware/components/settings_store/test/test_settings_store.c`
- `firmware/components/settings_app/settings_app.c`
- `firmware/components/diagnostics/resource_snapshot.c`

**步骤**

1. 定义带版本号和 CRC/值域校验的 NVS 配置。
2. 第一版包含定位开关/周期、服务地址和配置版本，不包含屏幕亮度。
3. 实现资源监控快照：SRAM、PSRAM、最大连续块、Flash/NVS/TF、页面/快照、任务栈和队列。TF 尚未挂载或不可用时显示 `N/A`，不能阻塞页面。
4. 资源页前台每秒刷新，离开页面停止计时器。
5. 测试缺失配置、损坏配置、版本迁移和写入失败。

### 任务 1.6：实现电量采集与设备状态

**新增文件**

- `firmware/components/battery_monitor/CMakeLists.txt`
- `firmware/components/battery_monitor/include/battery_monitor.h`
- `firmware/components/battery_monitor/battery_monitor.c`
- `firmware/components/battery_monitor/test/test_battery_state.c`

**步骤**

1. 根据开发板原理图确认 MAX17048 的实际总线、地址和告警连接，把结果写入 `board_config.h`；未确认前不假设它与触摸共享 GPIO15/16。
2. 周期读取电压、估算电量和芯片状态，转换为带时间戳的设备状态。
3. 读取失败保留最后有效值并标记过期，不能显示伪造的 0% 或 100%。
4. 将电量状态提供给状态栏、资源监控和后续设备状态上报。
5. 真机比较 MAX17048 读数与万用表电压，记录误差和低电量阈值。

**通过条件**

- 在 4G 不发射、负载稳定 60 秒后，于约 4.1V、3.8V、3.5V 三个电压点测量，MAX17048 与万用表误差均不超过 ±100mV。
- SOC 始终位于 0–100%，稳定负载下一分钟内不得无原因跳变超过 5 个百分点，受控放电过程中总体单调下降。
- 未完成完整充放电标定前不承诺绝对 SOC 精度；设置页可显示估算电量，但验证文档必须保留“估算值”说明。

### 阶段 1 验收

执行架构文档“桌面内核”全部验收项，并把结果写入 `docs/verification/phase-1-desktop.md`。阶段结束时运行：

```bash
cd firmware && idf.py build
cd test_apps/unit
idf.py set-target esp32s3
idf.py build
pytest --embedded-services esp,idf pytest_unit.py
```

通过条件：pytest 进程返回 0，Unity 汇总为 0 failures；只看到串口输出但未得到测试汇总不算通过。

## 阶段 2：电话服务与电话 APP

### 任务 2.1：实现 AT 解析器和唯一 Modem Manager

**新增文件**

- `firmware/components/modem_manager/include/modem_manager.h`
- `firmware/components/modem_manager/include/modem_request.h`
- `firmware/components/modem_manager/at_line_parser.c`
- `firmware/components/modem_manager/modem_manager.c`
- `firmware/components/modem_manager/test/test_at_line_parser.c`
- `firmware/components/modem_manager/test/test_modem_scheduler.c`

**步骤**

1. 用阶段 0 保存的真实串口样本先写解析测试。
2. 分离命令响应、最终结果、提示符和异步 URC。
3. 实现单事务调度器、优先级、超时、取消、清理窗口和代次编号。
4. 上传等普通请求可取消；通话控制和关键 URC 使用最高优先级。
5. 禁止其他组件链接或调用 UART API；通过组件依赖保持边界。

### 任务 2.2：实现 Call Service 双层状态机

**新增文件**

- `firmware/components/call_service/include/call_service.h`
- `firmware/components/call_service/include/call_state.h`
- `firmware/components/call_service/call_state_machine.c`
- `firmware/components/call_service/call_service.c`
- `firmware/components/call_service/test/test_call_state_machine.c`

**步骤**

1. 先覆盖模组 `Off/Booting/Registering/Ready/Error` 转换测试。
2. 覆盖呼叫 `Idle/Incoming/Dialing/Alerting/Active/Ending/Failed` 转换测试。
3. 映射忙线、拒接、无人接听、无服务、对方挂断、网络中断和命令错误。
4. 重复 URC 保持幂等；迟到响应不能改变新呼叫会话。
5. Modem Manager 重启后查询模组状态并恢复 Call Service。

### 任务 2.3：实现电话 APP 和系统来电覆盖层

**新增文件**

- `firmware/components/phone_app/phone_app.c`
- `firmware/components/phone_app/dialer_page.c`
- `firmware/components/phone_app/call_page.c`
- `firmware/components/phone_app/incoming_overlay.c`

**步骤**

1. 实现拨号、删除、呼叫、接听、拒接和挂断 UI。
2. 所有操作发送结构化 Call Service 命令，不直接发 AT。
3. 任意 APP 前台收到来电时显示系统覆盖层，并保存页面恢复点。
4. 通话中关闭电话任务卡只销毁 UI，会话继续；桌面状态入口可重建通话页。
5. 逐项完成真实网络异常验收并记录 URC 与最终 UI 状态。

### 阶段 2 验收

完成架构文档“功能集成与稳定性”中的电话项，结果写入 `docs/verification/phase-2-call.md`。最终 SIM 验证仍保留为上线门槛。

## 阶段 3：相机、相册与内存预算

### 任务 3.1：完成有界 Media Worker

**修改文件**

- `firmware/components/media_worker/include/media_job.h`
- `firmware/components/media_worker/media_worker.c`
- `firmware/components/media_worker/test/test_media_queue.c`

**步骤**

1. 定义拍摄、缩放、JPEG 解码、TF 读写任务和优先级。
2. 所有任务携带所有者、会话 ID、取消标记和唯一释放路径。
3. 队列满时用户任务返回“系统忙”，可替换的后台快照只保留最新任务。
4. 使用单帧 JPEG 缓冲起步，禁止相机和大图解码同时无限占用 PSRAM。

### 任务 3.2：实现相机 APP

**新增文件**

- `firmware/components/camera_app/camera_app.c`
- `firmware/components/camera_app/camera_preview.c`
- `firmware/components/camera_app/capture_flow.c`

**步骤**

1. 使用低分辨率预览，拍照时切换到默认 VGA JPEG。
2. 单张上限 200KB；超限按预设阶梯降低质量或分辨率重拍。
3. 照片使用临时文件写入、同步、原子改名，避免断电留下假成功文件。
4. 保存成功后立即归还相机帧缓冲，上传只读取 TF 文件。
5. 页面退后台后停止预览并让传感器休眠。
6. 真机连续预览 10 分钟，确认画面方向、颜色、刷新、退出和重新进入正常；预览期间底部系统手势仍可响应。

**通过条件**

- 使用 QVGA JPEG、320×240 或更小预览区域，10 分钟平均渲染帧率不低于 10 FPS。
- 不出现超过 1 秒且不能自行恢复的画面冻结；采集或解码失败率低于 1%。
- 连续退出并重新进入相机 20 次均能恢复预览，无持续内存下降。
- 预览期间底部系统手势到首个视觉反馈仍不高于 50ms。

### 任务 3.3：实现相册 APP

**新增文件**

- `firmware/components/gallery_app/gallery_app.c`
- `firmware/components/gallery_app/gallery_index.c`
- `firmware/components/gallery_app/image_decode.c`

**步骤**

1. 分页扫描照片元数据，不一次加载所有缩略图。
2. JPEG 解码和缩放在 Media Worker 中执行。
3. 删除照片采用明确确认和失败恢复。
4. 页面关闭时取消排队任务，迟到结果按会话 ID 丢弃。

### 任务 3.4：建立内存预算和安全线

**修改文件**

- `firmware/components/diagnostics/resource_snapshot.c`
- `docs/verification/phase-3-media-memory.md`

**步骤**

1. 阶段 3 记录空闲桌面、四任务快照、相机预览、拍照和相册解码场景；HTTPS 上传及媒体并发在 Network Worker 完成后的任务 4.5 补测。
2. 记录内部 SRAM/PSRAM 最低剩余、最大连续块、任务栈高水位和分配失败。
3. 根据数据设定内存安全线和拒绝新重任务阈值。
4. 阶段 3 只给出媒体侧临时安全线；在任务 4.5 的网络并发峰值完成前，后台页面缓存保持一个，不得提高到两个。

## 阶段 4：定位、网络和云端协议

### 任务 4.1：实现 GNSS 解析和 Location Service

**新增文件**

- `firmware/components/location_service/include/location_service.h`
- `firmware/components/location_service/gnss_parser.c`
- `firmware/components/location_service/location_state_machine.c`
- `firmware/components/location_service/location_service.c`
- `firmware/components/location_service/test/test_gnss_parser.c`
- `firmware/components/location_service/test/test_location_state.c`

**步骤**

1. 使用真实 `AT+CGNSSINFO` 样本覆盖有效、空字段、损坏和半包测试。
2. 正确转换 `ddmm.mmmm`，验证南纬和西经负号。
3. 实现 `Disabled/Starting/NoFix/Fixed/Stale/Error`。
4. 每个样本包含 ID、定位时间、接收时间和卫星信息。
5. 失锁后保留最后位置并标龄，不更新当前位置时间。

### 任务 4.2：实现固定 Network Worker

**新增文件**

- `firmware/components/network_worker/include/network_job.h`
- `firmware/components/network_worker/network_worker.c`
- `firmware/components/network_worker/https_client.c`
- `firmware/components/network_worker/test/test_network_queue.c`
- `firmware/components/network_worker/test/test_retry_policy.c`

**步骤**

1. 使用固定任务、有界队列和 8–16KB 上传缓冲。
2. 实现总超时、有上限指数退避、取消和来电抢占。
3. JSON 位置任务可合并，只保留最新未发送样本；照片任务保留明确文件所有权。
4. 网络链路断开时不持有相机帧缓冲，也不无限积压照片任务。

### 任务 4.3：定义设备云端 API 契约

**新增文件**

- `docs/api/device-cloud-api.md`
- `firmware/components/network_worker/api_codec.c`
- `firmware/components/network_worker/test/test_api_codec.c`

**契约内容**

1. 设备状态和位置上报请求/响应。
2. 照片分块或单请求上传规则、大小上限和内容摘要。
3. 样本 ID、请求 ID 和幂等键。
4. 错误码、重试语义、服务端时间校准和最大响应体。
5. 数据保留和照片访问授权要求。
6. 远程命令只在文档的“后续圆屏扩展”章节记录安全约束：唯一 ID、创建时间、过期时间、一次性消费和执行回执。本阶段不实现命令消费者，也不把它作为固件主线验收门槛。

**验证**

- API codec 使用固定 JSON 样本测试，不依赖真实网络。
- HTTPS client 通过可注入 transport 的 fake 覆盖成功、401、403、409、429、5xx、超时和断流。
- 蜂窝集成测试使用互联网可达的 staging HTTPS 端点，配置公开可信证书或专用测试 CA；不假设设备能访问开发机上的 localhost。
- 集成测试先通过模组时间或 SNTP 建立可信时间，再校验证书；禁止以关闭证书校验绕过时钟问题。

### 任务 4.4：实现设备凭据保护

**新增文件**

- `firmware/components/settings_store/credential_store.c`
- `firmware/components/settings_store/test/test_credential_store.c`
- `firmware/tools/provision_dev_device.py`

**步骤**

1. 启用 NVS Encryption 和 NVS keys 分区。
2. 开发设备通过本地 provisioning 工具生成独立测试凭据并写入加密 NVS；工具输入和生成物位于 Git 忽略目录，不把凭据提交到 Git 或打印到日志。
3. 定义空白设备、凭据损坏、开发恢复出厂和吊销后的行为：网络功能保持锁定并提示重新 provision，不回退到硬编码共享密钥。
4. 支持凭据版本、重新配置和轮换；开发凭据与未来生产凭据使用不同颁发环境。
5. HTTPS 必须校验服务端证书；无可信时间时先完成可信校时，不允许直接关闭证书校验。
6. 生产设备的安全注入、密钥托管和量产恢复流程不在本阶段实现，但列为发布阻断项。

### 任务 4.5：补齐网络并发峰值和最终内存安全线

**修改文件**

- `firmware/components/diagnostics/resource_snapshot.c`
- `docs/verification/phase-3-media-memory.md`
- `docs/verification/phase-4-location-network.md`

**步骤**

1. 测量 HTTPS 空闲、位置上报、200KB 照片上传、上传期间桌面切换和上传期间来电场景。
2. 与阶段 3 的相机预览、拍照、相册解码和任务快照组合，记录内部 SRAM/PSRAM 最低剩余、最大连续块和任务栈水位。
3. 固化最终内存安全线、Media/Network Worker 队列长度和拒绝新重任务阈值。
4. 只有全部媒体与网络并发峰值通过安全线，才评估后台页面缓存从一个提高到两个；没有明确收益则保持一个。

### 阶段 4 边界

阶段 4 只要求设备端通过 codec/fake transport 测试和互联网可达的 staging HTTPS 端点验收。正式云端后端、家属地图和远程命令消费者需要先确定技术栈、部署环境、地图供应商和账户入口，然后单独设计、计划和实施。

## 阶段 5：整机稳定性与发布

### 任务 5.1：故障注入和压力测试

**新增文件**

- `firmware/components/diagnostics/include/fault_injection.h`
- `firmware/components/diagnostics/fault_injection.c`
- `docs/verification/phase-5-stability.md`

**覆盖场景**

- LVGL Snapshot 分配失败。
- 普通事件队列拥堵和高优先级事件突发。
- Media/Network Worker 队列满。
- UART 半包、噪声、AT 超时和迟到响应。
- 网络链路断开、DNS 失败、TLS 失败和 HTTP 超时。
- GNSS 长时间无定位和模组重启。
- TF 卡拔出、文件写满和 JPEG 损坏。
- 相机初始化失败和拍照期间来电。

通过条件：系统按规格降级，无 UI 永久阻塞、任务栈溢出、重复释放或持续内存泄漏。

### 任务 5.2：两小时并发运行

**步骤**

1. 循环桌面翻页、APP 打开关闭、相机拍照、相册浏览、GNSS 查询和 HTTPS 上传。
2. 在循环中插入真实呼入、呼出和网络链路重连。
3. 每分钟记录资源快照和队列深度。
4. 比较开始与结束的可用堆、最大连续块和任务数量。

**通过条件**

- 无崩溃、看门狗、栈溢出或无法恢复的服务状态。
- 无持续单调下降且无法回收的内存趋势。
- 触摸和系统手势在重任务期间仍满足响应目标。

### 任务 5.3：形成发布基线

**修改/新增文件**

- `firmware/sdkconfig.defaults`
- `firmware/dependencies.lock`
- `docs/verification/phase-5-stability.md`
- `README.md`

**步骤**

1. 固化最终 SPI 时钟、任务栈、队列长度、内存安全线和分区表。
2. 记录所有硬件版本、组件版本、编译命令和烧录步骤。
3. 明确剩余上线门槛：最终 SIM/套餐/地区、正式云端、生产凭据注入与恢复流程和外壳散热。
4. 从干净克隆完成构建和烧录演练。

## 每个任务的完成定义

一个任务只有同时满足以下条件才算完成：

1. 对应代码和文档已落盘。
2. 新增逻辑有自动测试，硬件逻辑有可重复的真机步骤。
3. `idf.py build` 通过，没有新增编译警告。
4. 相关测试或真机验收通过，并记录实际输出。
5. 资源所有权、超时、取消和错误路径已验证。
6. 没有把密钥、SIM 信息、电话号码或服务凭据提交到仓库。

## 推荐执行批次

为降低返工，按以下批次执行并在每批结束时审阅：

1. 任务 0.1–0.3：工程骨架、依赖和诊断。
2. 任务 0.4–0.8：P0 硬件与启动验证。
3. 任务 1.1–1.6：桌面内核和设备状态。
4. 任务 2.1–2.3：电话系统。
5. 任务 3.1–3.4：媒体和内存预算。
6. 任务 4.1–4.5：定位、网络、安全协议和最终内存线。
7. 任务 5.1–5.3：稳定性和发布基线。

每个批次开始前，只展开该批次的任务；如果阶段门槛失败，先更新需求/架构和本计划，再决定是否继续。