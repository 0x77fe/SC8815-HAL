# 寄存器与 API 参考

## 寄存器表

下列地址和位域对应 [sc8815_reg.h](../sc8815_reg.h) 及 [驱动实现](../sc8815.cpp)。访问列区分配置寄存器、采样或状态寄存器以及保留地址。

| 地址 | 寄存器 | 访问 | 用途 |
| --- | --- | --- | --- |
| `0x00` | `VBAT_SET` | R/W | VBAT 充电目标电压 / 电池配置 |
| `0x01` | `VBUSREF_I_SET` | R/W | VBUS 内部参考电压 [9:2] (FB_SEL=0) |
| `0x02` | `VBUSREF_I_SET2` | R/W | VBUS 内部参考电压 [1:0] 位于寄存器 [7:6]，低 6 位保留 |
| `0x03` | `VBUSREF_E_SET` | R/W | VBUS 外部参考电压 [9:2] (FB_SEL=1) |
| `0x04` | `VBUSREF_E_SET2` | R/W | VBUS 外部参考电压 [1:0] 位于寄存器 [7:6]，低 6 位保留 |
| `0x05` | `IBUS_LIM_SET` | R/W | IBUS 电流限制 (充/放电均生效) |
| `0x06` | `IBAT_LIM_SET` | R/W | IBAT 电流限制 (充/放电均生效) |
| `0x07` | `VINREG_SET` | R/W | VINREG 动态电源管理参考电压 |
| `0x08` | `RATIO` | R/W | 电流/电压比率配置 (含 保留位) |
| `0x09` | `CTRL0_SET` | R/W | 控制0: EN_OTG, VINREG_RATIO, FREQ, DT |
| `0x0A` | `CTRL1_SET` | R/W | 控制1: FB_SEL, DIS_OVP, 充电相关 |
| `0x0B` | `CTRL2_SET` | R/W | 控制2: FACTORY, EN_DITHER, SLEW |
| `0x0C` | `CTRL3_SET` | R/W | 控制3: AD_START, EN_PFM, 保护配置 |
| `0x0D` | `VBUS_FB_VALUE` | R | VBUS ADC 值 [9:2] |
| `0x0E` | `VBUS_FB_VALUE2` | R | VBUS ADC 值 [1:0] 位于寄存器 [7:6]，低 6 位保留 |
| `0x0F` | `VBAT_FB_VALUE` | R | VBAT ADC 值 [9:2] |
| `0x10` | `VBAT_FB_VALUE2` | R | VBAT ADC 值 [1:0] 位于寄存器 [7:6]，低 6 位保留 |
| `0x11` | `IBUS_VALUE` | R | IBUS ADC 值 [9:2] |
| `0x12` | `IBUS_VALUE2` | R | IBUS ADC 值 [1:0] 位于寄存器 [7:6]，低 6 位保留 |
| `0x13` | `IBAT_VALUE` | R | IBAT ADC 值 [9:2] |
| `0x14` | `IBAT_VALUE2` | R | IBAT ADC 值 [1:0] 位于寄存器 [7:6]，低 6 位保留 |
| `0x15` | `ADIN_VALUE` | R | ADIN ADC 值 [9:2] |
| `0x16` | `ADIN_VALUE2` | R | ADIN ADC 值 [1:0] 位于寄存器 [7:6]，低 6 位保留 |
| `0x17` | `STATUS` | R | 中断状态 (AC_OK, INDET, VBUS_SHORT, OTP, EOC) |
| `0x18` | 保留 | — | 保留地址 |
| `0x19` | `MASK` | R/W | 中断掩码 |

## 函数参考

所有公开函数的首参均为 `const Sc8815 &chip`，用于提供七位地址、同步 I²C 传输函数和 RS1、RS2 阻值。返回 `true` 表示函数涉及的操作全部成功。传输失败或接口校验失败时返回 `false`。读函数的输出引用只在成功时使用。写函数通过 I²C 更新寄存器，输出为操作结果。

### `sc8815Begin`

- **输入** `chip`
- **输出** `bool` 操作结果

探测设备。以两个空缓冲调用传输函数，仅发送地址探测。

### `sc8815ReadRegister`

- **输入** `chip`、`reg`，寄存器地址
- **输出** `bool` 操作结果，成功时写入`data`，原始字节

读取一个寄存器字节。写入地址后重复起始读 1 字节，成功后赋值。

### `sc8815VbatSet`

- **输入** `chip`、`config`，`Sc8815VbatSetConfig`
- **输出** `bool` 操作结果

配置电池充电参数。将 IR 补偿、VBAT 选择、节数和单节目标电压写入 `0x00`。

### `sc8815Ratio`

- **输入** `chip`、`config`，`Sc8815RatioConfig`
- **输出** `bool` 操作结果

配置监测与限流倍率。打包四种倍率，读改写 `0x08` 并保留 [7:5]。

### `sc8815Ctrl0Set`

- **输入** `chip`、`config`，`Sc8815ModeAndSwitchingConfig`
- **输出** `bool` 操作结果

配置 OTG 与开关时序。编码 OTG、VINREG 倍率、频率与死区，读改写 `0x09` 并保留 [6:5]。

### `sc8815Ctrl1Set`

- **输入** `chip`、`config`，`Sc8815ChargingAndFeedbackConfig`
- **输出** `bool` 操作结果

配置充电、电压反馈与过压保护选项。编码电流基准、涓流、终止、反馈和 OVP 位，读改写 `0x0A` 并保留 [1:0]。

### `sc8815Ctrl2Set`

- **输入** `chip`、`config`，`Sc8815DitheringAndSlewRateConfig`
- **输出** `bool` 操作结果

配置工厂位、抖动和斜率。编码低 4 位，读改写 `0x0B` 并保留 [7:4]。

### `sc8815Ctrl3Set`

- **输入** `chip`、`config`，`Sc8815PowerPathAndMonitoringConfig`
- **输出** `bool` 操作结果

配置功率路径、ADC 和环路。将 PGATE、GPO、ADC、环路、折返、终止阈值和 PFM 打包后写入 `0x0C`。

### `sc8815VbusRefISet`

- **输入** `chip`、`ref_mv`，参考电压 mV
- **输出** `bool` 操作结果

设置内部 VBUS 参考。以 2 mV 步进计算 10 位编码，保留低寄存器低六位，从 `0x01` 连续写两字节。

### `sc8815VbusRefESet`

- **输入** `chip`、`ref_mv`，参考电压 mV
- **输出** `bool` 操作结果

设置外部 VBUS 参考。以同样的 10 位编码从 `0x03` 连续写两字节。调用方按外部反馈比分配参考电压。

### `sc8815IbusLimSet`

- **输入** `chip`、`current_a`，限流 A
- **输出** `bool` 操作结果

设置总线电流上限。从 `0x08` 读取 IBUS 倍率，以 RS1 计算 8 位编码并写 `0x05`。

### `sc8815IbatLimSet`

- **输入** `chip`、`current_a`，限流 A
- **输出** `bool` 操作结果

设置电池电流上限。从 `0x08` 读取 IBAT 倍率，以 RS2 计算 8 位编码并写 `0x06`。

### `sc8815VinregSet`

- **输入** `chip`、`voltage_mv`，参考电压 mV
- **输出** `bool` 操作结果

设置 VINREG 动态电源管理参考。从 `0x09` 读取倍率位，用 40 或 100 mV 步进计算 `电压 / 步进 - 1` 并写 `0x07`。

### `sc8815ReadVbusVoltage`

- **输入** `chip`
- **输出** `bool` 操作结果，成功时写入`voltage_mv`，mV

读取 VBUS 电压。从 `0x0D` 连续读两字节，按 `0x08` 的 VBUS 倍率选择 25 或 10 mV/步。

### `sc8815ReadVbatVoltage`

- **输入** `chip`
- **输出** `bool` 操作结果，成功时写入`voltage_mv`，mV

读取 VBAT 电压。从 `0x0F` 连续读两字节，按 `0x08` 的 VBAT 监测倍率选择 25 或 10 mV/步。

### `sc8815ReadIbusCurrent`

- **输入** `chip`
- **输出** `bool` 操作结果，成功时写入`current_a`，A

读取 IBUS 电流。从 `0x11` 连续读两字节，按 `0x08` 的 IBUS 倍率与 RS1 换算。

### `sc8815ReadIbatCurrent`

- **输入** `chip`
- **输出** `bool` 操作结果，成功时写入`current_a`，A

读取 IBAT 电流。从 `0x13` 连续读两字节，按 `0x08` 的 IBAT 倍率与 RS2 换算。

### `sc8815ReadAdinVoltage`

- **输入** `chip`
- **输出** `bool` 操作结果，成功时写入`voltage_mv`，mV

读取 ADIN 电压。从 `0x15` 连续读两字节，按 2 mV/步换算。

### `sc8815GetStatus`

- **输入** `chip`
- **输出** `bool` 操作结果，成功时写入`status`，`Sc8815Status`

读取中断状态。读 `0x17` 并拆出 AC_OK、INDET、VBUS_SHORT、OTP、EOC。读取会清除 INDET。

### `sc8815SetIrqMask`

- **输入** `chip`、`config`，`Sc8815IrqMaskConfig`
- **输出** `bool` 操作结果

写入中断掩码。读改写 `0x19`，保留 [7] 和 [4]，将 [0] 置 1。

### 编码与换算

- VBUS 参考编码为 `(ref_mv / 2 - 1) & 0x03FF`。按反馈电路及芯片工作范围选择参考电压。为 VBUS 参考提供 2 mV 的整数倍，为 VINREG 提供对应步进的整数倍，并确认寄存器编码可表示该设定。
- 电流限值计算 `current_a × 256 × Rs_mΩ / (ratio × 10) - 1`，加 0.5 后向零截断，写入一个字节。对应寄存器的物理限值为 `(code + 1) × ratio × 10 / (256 × Rs_mΩ)`。
- ADC 步数为 `(high << 2) + (low >> 6) + 1`。电流换算为 `steps × (2 / 1200) × ratio × 10 / Rs_mΩ`。其中 IBUS 使用 RS1，IBAT 使用 RS2，阻值单位均为 mΩ。
