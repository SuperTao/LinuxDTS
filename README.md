## SDM450设备树学习

基于高通SDM450平台分析设备树的特性。

#### 参考文档

https://blog.csdn.net/radianceblau/article/details/70800076

https://blog.csdn.net/radianceblau/article/details/74722395

https://blog.csdn.net/radianceblau/article/details/76574727

https://blog.csdn.net/hm131415/article/details/54377549

#### 设备树文件说明 

* msm8953-pinctrl.dtsi

	设置引脚复用功能。

* msm8953-mtp.dtsi

	配置具体功能参数，例如gpio

* msm8953.dtsi

	配置uart, i2c, spi等。

#### 代码分析

* [gpio_key代码分析](./document/gpio_key分析.md)

* [I2C设备树分析](./document/I2C设备树分析.md)

* [SPI设备树分析](./document/SPI设备树分析.md)

* [UART设备树分析](./document/UART设备树分析.md)
