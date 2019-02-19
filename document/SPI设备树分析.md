#### 引脚复用

arch/arm/boot/dts/IDH60/msm8953-pinctrl.dtsi

```
        spi3 {
            spi3_default: spi3_default {
                /* active state */
                mux {
                    /* MOSI, MISO, CLK */
                    pins = "gpio8", "gpio9", "gpio11";	// gpio引脚
                    function = "blsp_spi3";				// 复用成spi
                };   

                config {
                    pins = "gpio8", "gpio9", "gpio11";
                    drive-strength = <12>; /* 12 MA */	// 电流
                    bias-disable = <0>; /* No PULL */	// 无上拉下拉
                };   
            };   

            spi3_sleep: spi3_sleep {
                /* suspended state */
                mux {
                    /* MOSI, MISO, CLK */
                    pins = "gpio8", "gpio9", "gpio11";
                    function = "gpio";				// suspend状态配置成gpio
                };   

                config {
                    pins = "gpio8", "gpio9", "gpio11";
                    drive-strength = <2>; /* 2 MA */	//suspend电流
                    bias-pull-down; /* PULL Down */		// 下拉
                };   
            };   

            spi3_cs0_active: cs0_active {		// spi片选引脚
                /* CS */
                mux {
                    pins = "gpio10";		// gpio
                    function = "blsp_spi3";	// 引脚复用
                };

                config {
                    pins = "gpio10";
                    drive-strength = <2>;
                    bias-disable = <0>;
                };
```				

#### SPI参数配置				

msm8953.dtsi

```	
	spi_3: spi@78b7000 { /* BLSP1 QUP3 */
        compatible = "qcom,spi-qup-v2";
        #address-cells = <1>; 
        #size-cells = <0>; 
        reg-names = "spi_physical", "spi_bam_physical";
        reg = <0x78b7000 0x600>,
            <0x7884000 0x1f000>;
        interrupt-names = "spi_irq", "spi_bam_irq";
        interrupts = <0 97 0>, <0 238 0>;		// 中断号
        spi-max-frequency = <19200000>;			// 频率
        pinctrl-names = "spi_default", "spi_sleep";	//所用到的pinctrl
        pinctrl-0 = <&spi3_default &spi3_cs0_active>;// active
        pinctrl-1 = <&spi3_sleep &spi3_cs0_sleep>;	// sleep
        clocks = <&clock_gcc clk_gcc_blsp1_ahb_clk>,
            <&clock_gcc clk_gcc_blsp1_qup3_spi_apps_clk>;
        clock-names = "iface_clk", "core_clk";
        qcom,infinite-mode = <0>; 
        qcom,use-bam;
        qcom,use-pinctrl;
        qcom,ver-reg-exists;
        qcom,bam-consumer-pipe-index = <8>; 
        qcom,bam-producer-pipe-index = <9>; 
        qcom,master-id = <86>;
        status = "disabled";
    };
```

#### SPI总线上的设备

msm8953-mtp.dtsi	

```
&spi_3 {  BLSP1 QUP3 
    spi-max-frequency = <16000000>;
    maxim_sti@0 {
        status = "disabled";
        compatible = "maxim,maxim_sti";
        reg = <0>;
        interrupt-parent = <&tlmm>;
        interrupts = <65 0>;
        spi-max-frequency = <16000000>;
        avdd-supply = <&pm8953_l10>;
        dvdd-supply = <&pm8953_l5>;
        maxim_sti,irq-gpio = <&tlmm 65 0x00>;
        maxim_sti,reset-gpio = <&tlmm 64 0x00>;
        maxim_sti,touch_fusion = "/vendor/bin/touch_fusion";
        maxim_sti,config_file = "/etc/firmware/qtc800s.cfg";
        maxim_sti,fw_name = "qtc800s.bin";
        pinctrl-names = "pmx_ts_active","pmx_ts_suspend",
                        "pmx_ts_release";
        pinctrl-0 = <&ts_int_active &ts_reset_active>;
        pinctrl-1 = <&ts_int_suspend &ts_reset_suspend>;
        pinctrl-2 = <&ts_release>;
    };
};
```
