---
layout: post
title: "Embedded Machine Learning - Part 3: Adding environmental sensors"
categories:
  - Projects
  - Electronics
---

To predict the weather, of course we need to feed live data to our model. Quantities that are relatively easy to measure and that may or may not help our model predict the weather in the future are:

- Humidity
- Pressure
- Temperature

All of these can be measured with very little overhead using cheaply available sensor. For this project, I will be using Infineon's [DPS310](https://www.infineon.com/dgdl/Infineon-DPS310-DataSheet-v01_01-EN.pdf?fileId=5546d462576f34750157750826c42242) for pressure and temperature as well as the [BME280](https://www.bosch-sensortec.com/products/environmental-sensors/humidity-sensors-bme280/) from Bosch for humidity. Both of the sensor can be connected to the PSoC6 using [I²C](https://en.wikipedia.org/wiki/I%C2%B2C).

## DPS310
For the DPS310, Infineon provides an [Arduino Library](https://github.com/Infineon/DPS310-Pressure-Sensor). In order to use that, however, one would at least have to port the `Wire` library - Arduino's low-level I2C implementation - over to the PSoC6. Instead of doing that, I went with the [Sensidev DPS310](https://github.com/sensidev/sensor-dps310) driver that I found on Github. It's written in plain C and can easily be compiled against a handful of self-implemented functions for I2C access.

Before we can use I2C, we need to initialize it, which CyHAL makes quite easy:

```c
#include "cyhal.h"
#include "cybsp.h"

cyhal_i2c_t i2c_master_obj;
const uint32_t I2C_MASTER_FREQUENCY = 400000u;
cyhal_i2c_cfg_t i2c_master_config = {CYHAL_I2C_MODE_MASTER, 0, I2C_MASTER_FREQUENCY};

void i2c_init(void) {
    cy_rslt_t result;
    result = cyhal_i2c_init(&i2c_master_obj, CYBSP_I2C_SDA, CYBSP_I2C_SCL , NULL);
    CY_ASSERT(result == CY_RSLT_SUCCESS);

    result = cyhal_i2c_configure(&i2c_master_obj, &i2c_master_config);
    CY_ASSERT(result == CY_RSLT_SUCCESS);
}
```

This initializes I2C to be on `P6.0` (SCL) and `P6.1` (SDA) of the CY8KIT-062. The pins are located in the same location as they would be for an Arduino Uno.

In order to use the DPS310 library, I2C read and write functions are required for the library to access the sensor. These functions will be simple wrappers that allow the library to access the CyHAL functions. The two sensors use memory-mapped I/O. This means that we can't just read from or write to them. Every operation must instead be preceded by a write access in which we tell the sensor *which* location in memory we would like to read or write. Luckily, this pattern is so common that CyHAL already implements it for us, so that we don't have to keep track of the individual operations. The functions that do this are `cyhal_i2c_master_mem_read()` and `cyhal_i2c_master_mem_write`. We just need to provide a simple wrapper that maps the call parameters to the format that the library expects.

```c
int8_t dps310_i2c_read(uint8_t address, uint8_t reg, uint8_t *data, uint16_t count) {
    cy_rslt_t result;
    result = cyhal_i2c_master_mem_read(&i2c_master_obj, address, reg, 1, data, count, 0);
    CY_ASSERT(result == CY_RSLT_SUCCESS);
    return DPS310_OK;
}

int8_t dps310_i2c_write(uint8_t address, uint8_t reg, const uint8_t *data, uint16_t count) {
    cy_rslt_t result;
    result = cyhal_i2c_master_mem_write(&i2c_master_obj, address, reg, count, data, count, 0);
    CY_ASSERT(result == CY_RSLT_SUCCESS);
    return DPS310_OK;
}

void dps310_i2c_delay_ms(uint32_t delay) {
    cyhal_system_delay_ms(delay);
}
```

With this in place, we first need to initialize the sensor

```c
int16_t dps310_init() {

    int16_t ret;

    ret = product_id_check();
    if (ret != DPS310_OK) return ret;
    
    ret = read_coefs();
    if (ret != DPS310_OK) return ret;

    dps310_configure_temperature(
            DPS310_CFG_RATE_1_MEAS |
            g_temperature_prc);

    dps310_configure_pressure(
            DPS310_CFG_RATE_1_MEAS |
            g_pressure_prc);

    return DPS310_OK;
}
```

and can then retrieve the values for temperature and pressure from it:

```c
float temperature = 0.0;
float pressure = 0.0;

dps310_read(&temperature, &pressure);
```

**Note** As of writing of this post, there is an issue with the library that is provided on Github as is, through which measurements are incorrect when the oversampling rate and sample frequency diverge (read the corresponding [Github issue](https://github.com/sensidev/sensor-dps310/issues/2) for more information)

## BME280
The implementation for the Bosch Sensor follows a similar pattern, but is a bit more straight-forward because Bosch provides a non-Arduino [reference library](https://github.com/BoschSensortec/BME280_driver). Parts of the usage guide in the repository seem to be outdated, though. Again, we need to provide wrappers around the same CyHAL functions as above so that the sensor driver can call them:

```c
int8_t bme280_i2c_read(uint8_t reg_addr, uint8_t *reg_data, uint32_t len, void *intf_ptr)
{
    uint16_t addr = (uint16_t) *((uint8_t *) intf_ptr);
    
    cy_rslt_t result;
    result = cyhal_i2c_master_mem_read(&i2c_master_obj, addr, reg_addr, 1, reg_data, len, 0);
    CY_ASSERT(result == CY_RSLT_SUCCESS);   
    return 0;
}

int8_t bme280_i2c_write(uint8_t reg_addr, uint8_t *reg_data, uint32_t len, void *intf_ptr)
{
    uint16_t addr = (uint16_t) *((uint8_t *) intf_ptr);
    cy_rslt_t result;
    result = cyhal_i2c_master_mem_write(&i2c_master_obj, addr, reg_addr, len, reg_data, len, 0);
    CY_ASSERT(result == CY_RSLT_SUCCESS);
    return 0;
}

// Note that this is microseconds, not milliseconds as the repository states
void bme280_delay_us(uint32_t period, void *intf_ptr)
{
    cyhal_system_delay_us(period);
}
```

with these wrappers in place, we can initialize the sensor as described in the repository

```c
struct bme280_dev bme280_dev;
struct bme280_data bme280_comp_data;
uint8_t bme280_addr = BME280_I2C_ADDR_PRIM;

void bme280_init(void) {
    bme280_dev.intf_ptr = &bme280_addr;
    bme280_dev.intf = BME280_I2C_INTF;
    bme280_dev.read = bme280_i2c_read;
    bme280_dev.write = bme280_i2c_write;
    bme280_dev.delay_us = bme280_delay_us;
    bme280_init(&bme280_dev);
}
```

we can then read from it using the read function that is provided in the repository

```c
int8_t stream_sensor_data_forced_mode(struct bme280_dev *dev)
{
    int8_t rslt;
    uint8_t settings_sel;

    dev->settings.osr_h = BME280_OVERSAMPLING_1X;
    dev->settings.osr_p = BME280_OVERSAMPLING_1X;
    dev->settings.osr_t = BME280_OVERSAMPLING_1X;
    dev->settings.filter = BME280_FILTER_COEFF_OFF;

    settings_sel = BME280_OSR_PRESS_SEL | BME280_OSR_TEMP_SEL | BME280_OSR_HUM_SEL;

    rslt = bme280_set_sensor_settings(settings_sel, dev);
    rslt = bme280_set_sensor_mode(BME280_FORCED_MODE, dev);
    rslt = bme280_get_sensor_data(BME280_ALL, &bme280_comp_data, dev);

    return rslt;
}
```

which can be called like this

```c
stream_sensor_data_forced_mode(&bme280_dev);
```

## What's next?
In order to clean this up a little bit, I have encapsulated the initialization of the sensors and the I2C peripheral in their own functions and moved them to a separate file. To make passing around results a bit easier, they can be stored in a struct like this

```c
typedef struct {
    float temperature;
    float pressure;
    float humidity;
} measurement_t;
```

I have also very much neglected proper error handling in most of the code. Implementing this would certainly be a good idea for any application that goes beyond a simple side project or prototype.
