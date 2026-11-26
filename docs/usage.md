# SC8815-HAL 使用指南

## 构建和集成

ESP-IDF 项目将本仓库放在 `components/sc8815`，并在调用方的 `idf_component_register()` 中添加 `PRIV_REQUIRES sc8815`。调用方对外头文件也包含 `sc8815.h` 时，使用 `REQUIRES sc8815`。

在独立 CMake 工程中，先引入仓库，再链接库目标。C++20 要求由库目标传递给调用方。

```cmake
add_subdirectory(SC8815-HAL)
target_link_libraries(your_app PRIVATE SC8815::HAL)
```

单独检查库的编译时运行 `cmake -S . -B build` 和 `cmake --build build`。交叉编译时为 CMake 提供目标平台的工具链文件。

## I²C 传输契约

构造 `Sc8815` 时传入同步传输函数的引用、七位地址及 RS1、RS2 的阻值。阻值以 mΩ 为单位，分别用于 IBUS 与 IBAT 的读数和限流换算。将七位地址按所用底层驱动的约定传入，默认地址见 `sc8815_i2c_address`。

| 写入缓冲 | 读取缓冲 | 回调行为 |
| --- | --- | --- |
| 空 | 空 | 探测指定地址，报告设备是否应答 |
| 有数据 | 空 | 将写入缓冲完整发送，首字节为寄存器地址 |
| 有数据 | 有长度 | 先发送寄存器地址，再以重复起始读取指定长度 |
| 空 | 有长度 | 执行纯读取事务 |

返回 `true` 表示整笔事务成功，NACK、超时或传输失败时返回 `false`。回调返回前完成传输，缓冲区的有效期覆盖本次调用。探测由 `sc8815Begin()` 完成。

ESP-IDF 的总线初始化、回调实现和采样循环见 [README 完整示例](../README.md#esp-idf-完整示例)。寄存器表和函数逐项说明见 [寄存器与 API 参考](reference.md)。

## 初始化及配置

1. 由板级程序控制 PSTOP，使功率级进入待机。
2. 初始化 I²C 并调用 `sc8815Begin()` 探测设备。
3. 根据电路选择反馈方式、电流检测倍率、工作模式和保护参数。按需启用 ADC。
4. 写入适用的参考电压和电流上限。确认所有调用成功后，由板级程序使能输出。

| 配置函数 | 主要职责 |
| --- | --- |
| `sc8815VbatSet` | 设置内阻补偿、VBAT 设定方式、串联节数及单节电压 |
| `sc8815Ratio` | 设置 IBAT、IBUS、VBAT 和 VBUS 检测倍率 |
| `sc8815Ctrl0Set` | 设置 OTG、VINREG 倍率、开关频率及死区 |
| `sc8815Ctrl1Set` | 设置电流检测、反馈来源、充电与过压保护选项 |
| `sc8815Ctrl2Set` | 设置工厂位、扩频抖动和参考电压变化斜率 |
| `sc8815Ctrl3Set` | 设置功率路径、ADC、控制环路和短路折返 |
| `sc8815SetIrqMask` | 配置芯片中断掩码 |

配置类型和取值见 [sc8815_reg.h](../sc8815_reg.h)。设置检测比例时选用枚举列出的有效值，并按实装电路填写采样电阻和反馈参数。

## 输出设定和测量

| 接口 | 参数或读数 |
| --- | --- |
| `sc8815VbusRefISet`、`sc8815VbusRefESet` | 传入所选反馈路径的参考电压，单位 mV，寄存器步进为 2 mV |
| `sc8815VinregSet` | 传入 VINREG 电压，单位 mV，步进取决于芯片的 40× 或 100× 倍率设置 |
| `sc8815IbusLimSet`、`sc8815IbatLimSet` | 传入电流上限，单位 A。驱动读取当前检测倍率，再结合 RS1 或 RS2 计算寄存器编码 |
| `sc8815ReadVbusVoltage`、`sc8815ReadVbatVoltage`、`sc8815ReadAdinVoltage` | 读取电压，单位 mV |
| `sc8815ReadIbusCurrent`、`sc8815ReadIbatCurrent` | 读取电流，单位 A |
| `sc8815ReadRegister` | 读取指定寄存器的一个原始字节 |
| `sc8815GetStatus` | 读取并解析状态寄存器。读取会清除 INDET |

按反馈网络和芯片允许范围计算参考电压，再传入满足步进的数值。外部反馈模式下，参考电压和输出端目标电压由电阻分压关系关联。调用前检查参考电压和 VINREG 的编码范围，驱动按寄存器位宽编码。读取 ADC 前通过 `sc8815Ctrl3Set()` 设置 `AdStart`，并等待板级电路与读数稳定。

## 错误处理与并发

每个接口返回 `bool`。仅在返回 `true` 时使用输出引用中的值。配置失败时由板级程序决定保持待机、记录错误或重试。

一次 HAL 调用可能执行多笔 I²C 事务，例如读取保留位后更新寄存器，或读取倍率后写入限流值。多任务共用芯片时，在调用方用同一把锁串行化完整 HAL 调用。多个设备共用物理总线时，再由总线驱动或板级传输层协调单笔事务。
