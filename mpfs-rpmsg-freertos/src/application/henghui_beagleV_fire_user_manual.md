
# BeagleV Fire — 工程变更与 IHC 兼容性说明

**变更摘要**
- 本文档汇总本工程与 BeagleV 提供工程相关的关键变更与注意事项，重点说明 IHC（Inter-Hart Communication）版本兼容性问题以及可行的应对方案。

**IHC 版本兼容性**
- 根据官方文档，PolarFire 的 IHC 在不同发行版中出现过版本变更（参见下方参考链接）。
- 本仓库自 `v2025.07` 及以后使用 IHC 2.0（IHC2.0）；而 `v2025.03` 及更早版本使用 IHC v0.6，两版本不兼容。
- Beagle 提供的工程中 FPGA IP 核版本为 `v0.6.0`。因此：
	- 如希望在当前（v2025.07+）的软件/工程上直接使用 Beagle 提供的 FPGA 设计，将会遇到不兼容问题；
	- 可选方案：替换 FPGA 设计中的 IHC IP 为 IHC2.0（并同步修改软件），或在软件层签出旧版本（v2025.03 或更早）以匹配 IP 核版本。

参考：polarfire IHC 文档

- https://github.com/polarfire-soc/polarfire-soc-documentation/blob/master/applications-and-demos/asymmetric-multiprocessing/ihc.md

**影响范围（典型）**
- FPGA 侧：IHC IP 核版本差异（寄存器映射、接口信号或参数变化）。
- 资源表（resource table / rsc_table）：设备地址、大小、IRQ 分配需要与 FPGA 设计保持一致。
- 驱动与中间件：`middleware/rpmsg`、platform 驱动、HAL 初始化代码可能需要适配新的 IHC 行为或寄存器定义。
- 板级配置与示例：`boards/*/fpga_design*`、platform 配置文件和示例工程。

**可行方案（概述）**
1. 降级软件以匹配 IP（快速、风险小）
	 - 在独立分支上签出或切换到旧标签（例如 v2025.03 或更早），以使用 IHC v0.6.0 对应的软件/驱动。
	 - 示例命令：

```
git fetch --all --tags
git checkout -b legacy-ihc v2025.03
```

2. 升级 FPGA IP 并适配软件（长期、工作量大）
	 - 在 FPGA 工具中将 IHC IP 更换为 IHC2.0，重新生成硬件描述。
	 - 更新资源表（`rsc_table`）以匹配新的地址/ IRQ/大小。
	 - 修改驱动与中间件（`middleware/rpmsg`、platform、hal）以适配新 IP 的寄存器/初始化流程。
	 - 重新生成 bitstream 并重新编译软件。

**建议的最小风险操作流程**
1. 先在隔离分支上尝试降级验证（见方案 1），确认示例（ping/pong、rpmsg demo）在旧分支能正常工作；
2. 若必须保留软件新分支，则按方案 2 在 FPGA 侧进行 IP 替换并逐步适配软件；
3. 对于 FPGA 设计变更，务必先备份原始 `fpga_design` 目录并建立回退分支。

**具体操作步骤（建议）**
- 备份 FPGA 设计目录：

```
cp -r mpfs-rpmsg-freertos/src/boards/beaglev-five/fpga_design \
	mpfs-rpmsg-freertos/src/boards/beaglev-five/fpga_design.bak
```

- 如选择降级软件：

```
git fetch --all --tags
git checkout -b legacy-ihc v2025.03
# 然后在该分支上构建并验证示例
cd mpfs-rpmsg-freertos/src
make clean && make all
```

- 如选择替换 IP：
	1. 在 FPGA 开发工具中将 IHC IP 替换为 IHC2.0，连线并生成新的硬件描述（bitstream / hdf）；
	2. 更新软件中的资源表（`rsc_table`）以反映新的基地址、IRQ 与长度；
	3. 针对 `middleware/rpmsg` 与 platform 驱动进行适配；
	4. 重新构建并在硬件上验证。

**测试与验证（端到端）**
- 验证点：rpmsg 通道是否能稳定建立，消息能否双向传递，中断是否正确触发，SMP/共享内存映射是否一致。
- 建议测试步骤：
	1. 编译 FPGA 并下载 bitstream（若替换 IP）；
	2. 在 target 上编译运行示例（pingpong、sample_echo_demo 等）；
	3. 使用串口/日志验证消息收发与异常信息；
	4. 若失败，比较旧/新 `rsc_table`、驱动初始化序列与寄存器映射差异。

示例（软件构建）命令：

```
cd mpfs-rpmsg-freertos/src
make clean && make all
```

**关键文件与位置（参考）**
- 文档与示例：`mpfs-rpmsg-freertos/src/application/henghui_beagleV_fire_user_manual.md`（本文件）
- 板级 FPGA 设计：`mpfs-rpmsg-freertos/src/boards/beaglev-five/`（fpga_design 相关子目录）
- 资源表与示例：`mpfs-rpmsg-freertos/src/application/inc/`（如 rsc_table.c / rsc_table.h）
- 中间件与驱动：`mpfs-rpmsg-freertos/src/middleware/`、`mpfs-rpmsg-freertos/src/platform/`。

**变更历史（建议格式，供团队记录）**

| 日期 | 组件 | 变更描述 | 影响范围 | 操作/回退 |
| ---- | ---- | -------- | -------- | -------- |
| 2026-07-09 | 文档 | 添加 IHC 兼容性与建议流程 | 所有 | 记录 |

----

**变更分析 — 逐文件说明（2026-07-09）**

以下为对工作区检测到的未提交修改的逐文件分析、可能影响与建议。

- `mpfs-rpmsg-bm/src/application/inc/demo_main.c`
	- 变更点：在 `start_demo()` 内，`mss_config_clk_rst` 的外设由 `MSS_PERIPH_MMUART1` -> `MSS_PERIPH_MMUART2`（仅在 `RPMSG_MASTER` 路径）。
	- 影响：BM（bare-metal）端的控制台/串口会切换到不同的 MMUART，需确认硬件连线与终端配置。
	- 建议：验证串口输出、同时检查 board 的 UART 引脚映射文档。

- `mpfs-rpmsg-bm/src/application/rules.mk`
	- 变更点：加入 `$(info ...)` 打印以表明 `MASTER` / `REMOTE` 配置（构建期输出）。
	- 影响：无功能性变化，只在 `make` 时增加可读性日志。
	- 建议：保持或去除信息打印，视 CI/构建日志清晰度需求而定。

- `mpfs-rpmsg-freertos/src/application/hart1/u54_1.c` 及 `hart2/u54_2.c`、`hart3/u54_3.c`、`hart4/u54_4.c`
	- 变更点：将 `start_demo()` 的调用从无参改为 `start_demo(hartid)`，以传入当前 hart id。
	- 影响：表明 `start_demo` 行为变为基于 hart 的差异化启动；需要同步更新函数声明/实现（已在 `demo_main.c` 更新）。
	- 建议：一次性编译验证四个 hart 的行为并检查各自串口消息。

- `mpfs-rpmsg-freertos/src/application/inc/demo_main.c`
	- 主要变更点：
		- 将 `start_demo()` 原型改为 `start_demo(uint64_t hartid)`，并在启动时输出 `"Demo Start in hart N"`。
		- 新增 keepalive 计时器与任务：`freertos_task_keepalive` 与 `keepalive_timer_cb`，每秒向串口输出心跳。
		- 修改了 `mss_config_clk_rst` 的 UART 选择（master/remote 路径的 MMUART 分配调整）。
	- 影响：
		- 引入额外任务与定时器会增加调度与堆栈/堆使用量，可能需调整 FreeRTOS 配置（堆大小/任务栈）。
		- 硬件串口使用变化需要同步检查板级映射与外设初始化。
		- 增强运行时可观测性（心跳、malloc 检查、rx 打印），有助于调试初始化或通信问题。
	- 建议：
		- 在目标板上做端到端验证：观察串口日志，确认 keepalive 和 demo 输出；若出现内存不足或任务创建失败，适当增大堆或任务栈。
		- 将调试输出（心跳/malloc 检查）在稳定后移为可选编译开关以降低运行时开销。

- `mpfs-rpmsg-freertos/src/application/inc/demo_main.h`
	- 变更点：更新 `start_demo` 声明为 `void start_demo(uint64_t hartid);`；并调整 `UART_APP` 宏的映射（master/remote 使用不同 `g_mss_uartX_lo` 实例），同时加入了若干注释化的 `MPFS_HAL_FIRST_HART/ LAST_HART` 行。
	- 影响：头文件接口变更，所有引用该接口的源文件必须重新编译；UART 宏改变会影响所有基于 `UART_APP` 的输出。
	- 建议：同步检查全仓对 `UART_APP` 的使用并确保一致性。

- `mpfs-rpmsg-freertos/src/application/rules.mk`
	- 变更点：在 `MASTER` 条件下添加 `-DMPFS_HAL_FIRST_HART=1 -DMPFS_HAL_LAST_HART=4`，并在 `else` 分支也添加同样 `CORE_CFLAGS` 定义，同时插入 `$(info ...)` 日志。
	- 影响：编译期宏将强制指定 hart 范围（1..4），可能覆盖其他地方的配置。
	- 建议：确认这些宏不会与 boards/platform 配置产生冲突；若需要多目标构建，请改为通过外部变量控制。

- `resources/hss-payload-freertos_freertos.yaml`
	- 变更点：新增一行被注释的 `hart-entry-points` 配置（把 `u54_4` 的入口地址置为 `0x00000000` 的注释示例）。
	- 影响：配置中保留了备选入口点注释，供快速切换参考；不会在构建脚本中生效，除非取消注释。

- 未跟踪文件（概要）
	- 列表包含大量 `mpfs-rpmsg-freertos/src/boards/beaglev-five/fpga_design` 与 `fpga_design_config` 下的头文件与 XML、以及 `mpfs-rpmsg-freertos/src/boards/beaglev-five/Makefile`、payload 二进制和 `.vscode` 设置文件。
	- 说明：这些文件大多为 FPGA 设计导出文件或板级配置/自动生成输出（体积大、频繁变动）。
	- 建议：
		- 将自动生成的 FPGA 输出（bitstreams、.xml/.h/.jou/.log 等）添加到 `.gitignore`，避免将大量生成文件直接提交到主仓库；
		- 仅将必要的板级配置（自定义 Makefile、小范围的 platform_config 变更）纳入版本控制；将大文件放到外部制品仓库或单独的 design 分支。

**总体影响与下一步建议**
- 本次改动以增强运行时可观测性（串口日志、keepalive、malloc 自检）与多 hart 支持为主，同时对 UART/外设映射和编译宏进行了调整。大部分改动属于调试与平台适配：
	- 立即动作：在隔离分支上完整构建并在目标板上运行（观察串口输出与任务创建成功）；
	- 性能/资源注意：增加任务与定时器会消耗 RTOS 堆与栈，请根据观察到的任务创建失败或 malloc 报错调整 `configTOTAL_HEAP_SIZE` 与任务栈大小；
	- 代码管理：将变更提交到功能分支（`feature/ihc-debug` 或类似），同时为大量生成文件更新 `.gitignore`。

我可以代为：
- 生成 `.gitignore` 建议补丁并提交到分支；
- 在当前工作区创建一个对比分支并提交这些修改（或生成 patch 文件）以便代码审查；
- 运行一次本地 `make` 并将构建错误/警告回报给你（如果你允许我在当前环境执行构建）。



