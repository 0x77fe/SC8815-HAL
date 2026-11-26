# SC8815-HAL

SC8815 的 C++20 驱动。通过同步 I²C 回调探测芯片、配置工作模式和保护参数、设置电压参考与限流，并读取 ADC 和状态寄存器。

## 集成

- ESP-IDF：将仓库放入项目的 `components/sc8815`，在调用组件的依赖中加入 `sc8815`。
- CMake：用 `add_subdirectory(SC8815-HAL)` 引入库，将应用链接到 `SC8815::HAL`。
- 其他构建系统：编译 `sc8815.cpp`，向应用添加库根目录的头文件搜索路径，并启用 C++20。

## ESP-IDF 完整示例

在 ESP-IDF 工程中将本仓库放入 `components/sc8815`，将以下代码保存为 `main/main.cpp`。示例使用 GPIO 5 控制 PSTOP、GPIO 8/9 连接 SDA/SCL，I²C 总线使用外部上拉。按实装电路填写引脚、采样电阻、反馈方式和保护参数后再接负载。

示例使用两只 10 mΩ 采样电阻、外部反馈参考 800 mV 和 IBUS 限流 1 A。当反馈分压倍率为 13.5 时，目标 VBUS 为 10.8 V。上电时让 PSTOP 保持高电平，待配置成功再由程序拉低。

`main/CMakeLists.txt`：

```cmake
idf_component_register(SRCS main.cpp PRIV_REQUIRES sc8815 esp_driver_gpio esp_driver_i2c)
```

`main/main.cpp`：

```cpp
#include "driver/gpio.h"
#include "driver/i2c_master.h"
#include "esp_err.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "sc8815.h"

// 配置板级引脚和 I²C 设备句柄。
static constexpr gpio_num_t power_stop_pin = GPIO_NUM_5;
static i2c_master_bus_handle_t bus = nullptr;
static i2c_master_dev_handle_t device = nullptr;

/*
* 执行 SC8815 的同步 I²C 事务。
*/
static bool boardI2cTransfer(std::uint8_t address, std::span<const std::uint8_t> write,
                             std::span<std::uint8_t> read)
{
    // 探测使用地址，数据传输使用已注册的设备。
    if (address != sc8815_i2c_address) return false;
    constexpr int timeout_ms = 10;

    // 按缓冲区组合选择事务，寄存器读取使用重复起始。
    if (write.empty() && read.empty()) return i2c_master_probe(bus, address, timeout_ms) == ESP_OK;
    if (write.empty()) return i2c_master_receive(device, read.data(), read.size(), timeout_ms) == ESP_OK;
    if (read.empty()) return i2c_master_transmit(device, write.data(), write.size(), timeout_ms) == ESP_OK;
    return i2c_master_transmit_receive(device, write.data(), write.size(),
                                       read.data(), read.size(), timeout_ms) == ESP_OK;
}

/*
* 初始化总线、配置 SC8815，并周期读取输出电压和电流。
*/
extern "C" void app_main()
{
    // 先将功率级置于待机。
    ESP_ERROR_CHECK(gpio_set_level(power_stop_pin, 1));
    ESP_ERROR_CHECK(gpio_set_direction(power_stop_pin, GPIO_MODE_OUTPUT));

    // 建立 I²C 总线并注册七位地址设备。
    i2c_master_bus_config_t bus_config{};
    bus_config.i2c_port = I2C_NUM_0;
    bus_config.sda_io_num = GPIO_NUM_8;
    bus_config.scl_io_num = GPIO_NUM_9;
    bus_config.clk_source = I2C_CLK_SRC_DEFAULT;
    bus_config.glitch_ignore_cnt = 7;
    ESP_ERROR_CHECK(i2c_new_master_bus(&bus_config, &bus));

    i2c_device_config_t device_config{};
    device_config.dev_addr_length = I2C_ADDR_BIT_LEN_7;
    device_config.device_address = sc8815_i2c_address;
    device_config.scl_speed_hz = 400000;
    ESP_ERROR_CHECK(i2c_master_bus_add_device(bus, &device_config, &device));

    // 绑定采样电阻并配置 OTG、外部反馈及 ADC。
    const Sc8815 chip{boardI2cTransfer, sc8815_i2c_address, 10.0f, 10.0f};
    const bool configured =
        sc8815Begin(chip) &&
        sc8815ConfigureBatteryCharging(chip,
            {SC8815_IRCOMP_0mOhm, SC8815_VBAT_SEL_EXTERNAL, SC8815_CSEL_1S, SC8815_VCELL_4V20}) &&
        sc8815ConfigureMeasurementAndLimitRatios(chip,
            {SC8815_IBAT_RATIO_6x, SC8815_IBUS_RATIO_6x, SC8815_VBAT_MON_RATIO_12_5x, SC8815_VBUS_RATIO_12_5x}) &&
        sc8815ConfigureModeAndSwitching(chip,
            {SC8815_EN_OTG_ENABLE, SC8815_VINREG_RATIO_100x, SC8815_FREQ_300kHz, SC8815_DT_20ns}) &&
        sc8815ConfigureChargingAndFeedback(chip,
            {SC8815_ICHAR_SEL_IBUS, SC8815_DIS_TRICKLE_ENABLE, SC8815_DIS_TERM_ENABLE,
             SC8815_FB_SEL_EXTERNAL, SC8815_TRICKLE_SET_70, SC8815_DIS_OVP_DISABLE}) &&
        sc8815ConfigureDitheringAndSlewRate(chip,
            {SC8815_FACTORY_POWERUP, SC8815_EN_DITHER_DISABLE, SC8815_SLEW_SET_2mV_us}) &&
        sc8815ConfigurePowerPathAndMonitoring(chip,
            {SC8815_EN_PGATE_OFF, SC8815_GPO_CTRL_OD, SC8815_AD_START_ENABLE, SC8815_ILIM_BW_SEL_5K,
             SC8815_LOOP_SET_IMPROVE, SC8815_DIS_SHORT_FOLDBACK_ENABLE, SC8815_EOC_SET_1_10, SC8815_EN_PFM_DISABLE}) &&
        sc8815SetIrqMask(chip, {}) &&
        sc8815SetVbusRefE(chip, 800) &&
        sc8815SetIbusLimit(chip, 1.0f);
    if (!configured)
    {
        ESP_LOGE("sc8815", "Configuration failed");
        return;
    }

    // 所有配置成功后使能输出，每 500 ms 读取一次。
    ESP_ERROR_CHECK(gpio_set_level(power_stop_pin, 0));
    for (;;)
    {
        vTaskDelay(pdMS_TO_TICKS(500));
        std::uint16_t voltage_mv;
        float current_a;
        if (sc8815ReadVbusVoltage(chip, voltage_mv) && sc8815ReadIbusCurrent(chip, current_a))
        {
            ESP_LOGI("sc8815", "VBUS=%u mV, IBUS=%.3f A", static_cast<unsigned>(voltage_mv), current_a);
        }
        else ESP_LOGW("sc8815", "ADC read failed");
    }
}
```

在 ESP-IDF 环境中选择目标芯片并构建。将 `PORT` 换成板卡串口。

```sh
idf.py set-target esp32c3
idf.py build
idf.py -p PORT flash monitor
```

[详细用法](docs/usage.md) · [寄存器与 API 参考](docs/reference.md) · [GPL-3.0-only](LICENSE)
