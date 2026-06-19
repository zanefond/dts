# RK3566 EVB1 DDR4 V10 移植 Armbian 完整记录

## 项目概述

将 Armbian 移植到 Rockchip RK3566 EVB1 DDR4 V10 开发板，基于 [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian) 项目。

### 硬件配置
| 组件 | 型号 |
|------|------|
| SoC | Rockchip RK3566 |
| PMIC | RK809 + TCS4525 (vdd_cpu) |
| 内存 | 2GB DDR4 |
| 存储 | eMMC、SD 卡、SPI NAND |
| 显示 | HDMI、eDP 面板 |
| WiFi/BT | RTL8821CS (SDIO + UART) |
| 以太网 | RTL8111H (PCIe) |
| 音频 | RK809 Codec、HDMI 音频、SPDIF |
| USB | USB3 OTG、USB3 Host、2×USB2.0 Host |
| RTC | HYM8563 |

### 涉及仓库
| 仓库 | 用途 |
|------|------|
| [zanefond/dts](https://github.com/zanefond/dts) | DTS 源文件 + CI 编译 DTB |
| [zanefond/amlogic-s9xxx-armbian](https://github.com/zanefond/amlogic-s9xxx-armbian) | Armbian 构建项目（fork 自 ophub） |
| [zanefond/kernel](https://github.com/zanefond/kernel) | 内核配置文件（fork 自 ophub/kernel） |

---

## Phase 1：DTB 硬件配置改进

### 文件：`rk3566-evb1-ddr4-v10.dts`

#### 1.1 WiFi (RTL8821CS)

```dts
&sdmmc1 {
    max-frequency = <50000000>;
    supports-sdio;
    bus-width = <4>;
    disable-wp;
    cap-sd-highspeed;
    cap-sdio-irq;
    keep-power-in-suspend;
    mmc-pwrseq = <&sdio_pwrseq>;
    non-removable;
    pinctrl-names = "default";
    pinctrl-0 = <&sdmmc1_bus4 &sdmmc1_cmd &sdmmc1_clk>;
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    sdio_wifi: wifi@1 {
        compatible = "realtek,rtl8821cs";
        reg = <1>;
        interrupt-parent = <&gpio2>;
        interrupts = <RK_PB2 IRQ_TYPE_LEVEL_LOW>;
        pinctrl-names = "default";
        pinctrl-0 = <&wifi_host_wake_irq>;
        wakeup-source;
        keep-power-in-suspend;
    };
};
```

#### 1.2 Bluetooth (RTL8821CS)

```dts
&uart1 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart1m0_xfer &uart1m0_ctsn>;
    status = "okay";
    uart-has-rtscts;

    bluetooth {
        compatible = "realtek,rtl8821cs-bt";
        clocks = <&rk809 1>;
        clock-names = "lpo";
        device-wake-gpios = <&gpio2 RK_PB4 GPIO_ACTIVE_HIGH>;
        host-wake-gpios = <&gpio2 RK_PB5 GPIO_ACTIVE_HIGH>;
        enable-gpios = <&gpio2 RK_PB1 GPIO_ACTIVE_HIGH>;
        pinctrl-names = "default";
        pinctrl-0 = <&uart1_gpios>;
        vbat-supply = <&vcc_wifi>;
        vddio-supply = <&vccio_sd>;
    };
};
```

#### 1.3 PCIe 电源控制

```dts
vcc3v3_pcie: vcc3v3-pcie-regulator {
    compatible = "regulator-fixed";
    regulator-name = "vcc3v3_pcie";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    enable-active-high;
    gpio = <&gpio0 RK_PC7 GPIO_ACTIVE_HIGH>;
    startup-delay-us = <100000>;
    vin-supply = <&vcc5v0_sys>;
    pinctrl-names = "default";
    pinctrl-0 = <&pcie_pwr_en>;
};

&pcie2x1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pcie20m1_pins>;
    reset-gpios = <&gpio4 RK_PD1 GPIO_ACTIVE_HIGH>;
    vpcie3v3-supply = <&vcc3v3_pcie>;
    status = "okay";
};
```

#### 1.4 CPU OPP 表

```dts
&cpu0_opp_table {
    compatible = "operating-points-v2";
    opp-shared;

    opp-408000000 {
        opp-hz = /bits/ 64 <408000000>;
        opp-microvolt = <900000 900000 1150000>;
        clock-latency-ns = <40000>;
    };
    // ... 600MHz, 816MHz, 1104MHz, 1416MHz
    opp-1608000000 {
        opp-hz = /bits/ 64 <1608000000>;
        opp-microvolt = <975000 975000 1150000>;
    };
    opp-1800000000 {
        opp-hz = /bits/ 64 <1800000000>;
        opp-microvolt = <1050000 1050000 1150000>;
    };
    opp-1992000000 {
        opp-hz = /bits/ 64 <1992000000>;
        opp-microvolt = <1150000 1150000 1150000>;
    };
};

&cpu0 { cpu-supply = <&vdd_cpu>; };
&cpu1 { cpu-supply = <&vdd_cpu>; };
&cpu2 { cpu-supply = <&vdd_cpu>; };
&cpu3 { cpu-supply = <&vdd_cpu>; };
```

#### 1.5 DSI 节点处理

```dts
&dsi0 {
    status = "disabled";
};

&dsi1 {
    status = "disabled";
};
```

> **注意**：不能直接 `/delete-node/ &dsi0`，因为 VOP 和 display-subsystem 内部通过 phandle 引用了 DSI endpoint。

---

## Phase 2：CI/CD 编译 DTB

### workflow：`.github/workflows/build-dtb.yml`

```yaml
name: Build RK3566 EVB1 DTB
on:
  push:
    paths:
      - 'rk3566-evb1-ddr4-v10.dts'
      - '.github/workflows/build-dtb.yml'
  pull_request:
  workflow_dispatch:

jobs:
  build-dtb:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - name: Checkout DTS repository
        uses: actions/checkout@v4

      - name: Clone Rockchip BSP kernel (shallow)
        run: |
          git clone --depth=1 --branch=develop-6.1 \
            https://github.com/rockchip-linux/kernel.git \
            linux-kernel

      - name: Copy DTS to kernel tree
        run: |
          cp rk3566-evb1-ddr4-v10.dts \
             linux-kernel/arch/arm64/boot/dts/rockchip/

      - name: Build DTB
        run: |
          cd linux-kernel
          make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
          make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
            DTC_FLAGS="-Wno-phandle_references" \
            rockchip/rk3566-evb1-ddr4-v10.dtb

      - name: Upload DTB artifact
        uses: actions/upload-artifact@v4
        with:
          name: rk3566-evb1-ddr4-v10-dtb
          path: |
            rk3566-evb1-ddr4-v10.dts
            linux-kernel/arch/arm64/boot/dts/rockchip/rk3566-evb1-ddr4-v10.dtb
```

### 关键问题 & 解决方案

| 问题 | 原因 | 解决 |
|------|------|------|
| DTB 只有 34KB | 直接 `dtc` 编译，无法解析 `#include "rk3566.dtsi"` | Clone Rockchip BSP 内核树，用 make 系统完整编译 |
| 正确的 DTB | 编译后 **99,828 字节** | **✅ 已解决** |
| `pcie20m1_pins` 重复标签 | 内核自带此 pinctrl 定义 | 删除自定义定义，直接使用内核默认 |
| `opp_table0` 不存在 | Rockchip BSP 内核无此标签 | 删除 `/delete-node/ &opp_table0` 块 |
| VOP-DSI phandle 警告 | `rk356x.dtsi` 中 display-subsystem 引用了 DSI endpoint | 添加 `DTC_FLAGS="-Wno-phandle_references"` |

---

## Phase 3：内核 DSI 驱动禁用

### 问题诊断

DSI 即使 DTB 中 `status = "disabled"`，built-in 驱动（`CONFIG_ROCKCHIP_DW_MIPI_DSI=y`）仍通过 DRM 框架绑定，导致无限重试循环：

```
dw-mipi-dsi-rockchip fe060000.dsi: failed to find panel or bridge: -517
rockchip-drm display-subsystem: bound fe040000.vop
dwhdmi-rockchip fe0a0000.hdmi: Detected HDMI TX controller v2.11a
rockchip-drm display-subsystem: bound fe0a0000.hdmi
dw-mipi-dsi-rockchip fe060000.dsi: failed to find panel or bridge: -517
→ 循环重启
```

### 修改文件

**仓库**：`zanefond/kernel` → `kernel-config/release/rk35xx/config-6.1`

| 行号 | 修改前 | 修改后 |
|------|--------|--------|
| 5561 | `CONFIG_DRM_MIPI_DSI=y` | `# CONFIG_DRM_MIPI_DSI is not set` |
| 5619 | `CONFIG_ROCKCHIP_DW_MIPI_DSI=y` | `# CONFIG_ROCKCHIP_DW_MIPI_DSI is not set` |
| 5620 | `CONFIG_ROCKCHIP_DW_MIPI_DSI2=y` | `# CONFIG_ROCKCHIP_DW_MIPI_DSI2 is not set` |
| 5782 | `CONFIG_DRM_DW_MIPI_DSI=y` | `# CONFIG_DRM_DW_MIPI_DSI is not set` |

### Workflow 更新

三个 workflow 的 `kernel_repo` 默认值已改为 `zanefond/kernel`：

| workflow | 文件 |
|----------|------|
| Build Armbian arm64 server image | `build-armbian-arm64-server-image.yml` |
| Build Armbian using releases files | `build-armbian-using-releases-files.yml` |
| Build Armbian from official images | `build-armbian-using-official-image.yml` |

---

## Phase 4：编译并部署

### 步骤 4.1：编译内核

1. 打开 https://github.com/zanefond/kernel/actions
2. 选择 **"Compile rockchip rk35xx kernel"** → **"Run workflow"**
3. 参数：
   | 参数 | 值 |
   |------|-----|
   | `kernel_source` | `unifreq/linux-6.1.y-rockchip` |
   | `kernel_version` | `6.1.y` |
   | `kernel_auto` | `true` |
4. 编译产物自动上传到 `zanefond/kernel` 的 Release

### 步骤 4.2：Build Armbian 镜像

1. 打开 https://github.com/zanefond/amlogic-s9xxx-armbian/actions
2. 选择 **"Build Armbian using releases files"** → **"Run workflow"**
3. 参数：
   | 参数 | 值 |
   |------|-----|
   | `set_release` | `bookworm` |
   | `armbian_board` | `rk3566-evb1-ddr4-v10` |
   | `armbian_kernel` | `6.1.y` |
   | `kernel_repo` | `zanefond/kernel` |
4. 镜像自动发布到 Release

---

## U-Boot 启动命令

进入 U-Boot 控制台后（开机按 Ctrl+C 中断 Android U-Boot）：

```
# 加载内核
load mmc 1:1 0x00200000 Image

# 加载板子专用 DTB
load mmc 1:1 0x08300000 dtb/rockchip/rk3566-evb1-ddr4-v10.dtb

# 设置内核启动参数
setenv bootargs earlycon=uart8250,mmio32,0xfe660000 console=ttyS2,1500000n8 root=/dev/mmcblk1p2 rw rootwait

# 启动
booti 0x00200000 - 0x08300000
```

### 参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| `mmc 1:1` | SD 卡第一分区 | boot 分区 (mmc 0 = eMMC) |
| `0x00200000` | RAM 地址 | Image 加载地址 |
| `0x08300000` | RAM 地址 | DTB 加载地址 |
| `ttyS2` | UART2 | 调试串口 (1500000 baud 8N1) |
| `/dev/mmcblk1p2` | SD 卡第二分区 | rootfs (ext4) |

---

## 验证命令

启动后执行：

```bash
# 检查 DSI 循环是否消失
dmesg | grep -i "dsi\|-517"

# 检查 WiFi/BT
dmesg | grep -i "sdio\|mmc1\|wifi\|bluetooth\|8821"

# 检查 PCIe 以太网
dmesg | grep -i "pcie\|r8125\|r8168\|r8169"

# 检查 CPU 频率
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

# 检查温度
cat /sys/class/thermal/thermal_zone0/temp
```

---

## 提交历史

### zanefond/dts
```
02c4396 fix: suppress phandle_references warnings for VOP-DSI cross-references
36dbe0c Phase 1: DTB improvements - WiFi/BT RTL8821CS, PCIe power/GPIO, CPU OPP table
8c70981 fix: disable unused DSI nodes to prevent kernel probe loop
```

### zanefond/amlogic-s9xxx-armbian
```
b846e3a switch default kernel_repo to zanefond/kernel
51c0618 update DTB: latest CI build with WiFi/BT, PCIe GPIO, CPU OPP
```

### zanefond/kernel
```
fbaf13f disable MIPI DSI drivers to fix RK3566 DRM probe loop
```

---

## 状态总览

| 阶段 | 状态 | 说明 |
|------|------|------|
| Phase 1: DTB 改进 | ✅ 完成 | WiFi/BT/PCIe/OPP 配置，已推送 |
| Phase 2: CI/CD | ✅ 完成 | build-dtb.yml 编译 DTB，上传 artifact |
| Phase 3: 内核配置 | ✅ 完成 | DSI 驱动已禁用，已推送至 zanefond/kernel |
| Phase 3: Workflow | ✅ 完成 | 3 个 workflow 已指向 zanefond/kernel |
| Phase 4: 编译内核 | 🔲 待办 | 在 zanefond/kernel 触发 "Compile rockchip rk35xx kernel" |
| Phase 4: Build 镜像 | 🔲 待办 | 内核编译完成后触发 "Build Armbian using releases files" |