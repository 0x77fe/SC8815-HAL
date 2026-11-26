# SC8815-HAL

SC8815 的 C++20 驱动。通过同步 I²C 回调探测芯片、配置工作模式和保护参数、设置电压参考与限流，并读取 ADC 和状态寄存器。

## 集成

- ESP-IDF：将仓库放入项目的 `components/sc8815`，在调用组件的依赖中加入 `sc8815`。
- CMake：用 `add_subdirectory(SC8815-HAL)` 引入库，将应用链接到 `SC8815::HAL`。
- 其他构建系统：编译 `sc8815.cpp`，向应用添加库根目录的头文件搜索路径，并启用 C++20。

## 示例

先初始化板级 I²C 总线，实现 `boardI2cTransfer()`，再调用 `readSc8815Vbat()`。示例使用七位地址 `0x74` 和两只 10 mΩ 采样电阻，请按电路填写采样电阻。

```cpp
#include "sc8815.h"

/*
* 执行板级同步 I²C 事务。
*/
bool boardI2cTransfer(std::uint8_t address, std::span<const std::uint8_t> write,
                      std::span<std::uint8_t> read);

/*
* 探测芯片并读取已启用 ADC 的 VBAT 电压。
*/
bool readSc8815Vbat(std::uint16_t &voltage_mv)
{
    // 绑定板级 I²C 传输及电流采样电阻。
    const Sc8815 chip{boardI2cTransfer, sc8815_i2c_address, 10.0f, 10.0f};

    // 探测成功后读取电压，单位为毫伏。
    return sc8815Begin(chip) && sc8815ReadVbatVoltage(chip, voltage_mv);
}
```

[详细用法](docs/usage.md) · [GPL-3.0-only](LICENSE)
