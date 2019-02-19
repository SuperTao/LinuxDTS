#### 引脚复用配置
arch/arm/boot/dts/IDH60/msm8953-pinctrl.dtsi
```
&soc {
	tlmm: pinctrl@1000000 {
		compatible = "qcom,msm8953-pinctrl";
		reg = <0x1000000 0x300000>;
		interrupts = <0 208 0>;
		gpio-controller;
		#gpio-cells = <2>;
		interrupt-controller;
		#interrupt-cells = <2>;
		[...省略...]
		
 		i2c_2 {
			i2c_2_active: i2c_2_active {			// active状态
				/* active state */
				mux {
					pins = "gpio6", "gpio7";		// I2C的SDA和SCL引脚
					function = "blsp_i2c2";			// 配置成i2c功能
				};

				config {
					pins = "gpio6", "gpio7";
					drive-strength = <2>;		// 设置驱动能力为2MA
					bias-disable;				// 没有内部上拉或者下拉
				};
			};

			i2c_2_sleep: i2c_2_sleep {			// sleep状态
				/* suspended state */
				mux {
					pins = "gpio6", "gpio7";
					function = "gpio";			// 配置成GPIO
				};

				config {
					pins = "gpio6", "gpio7";
					drive-strength = <2>;		// 电流
					bias-disable;				// 无上拉下拉
				};
			};
		};

		i2c_3 {
			i2c_3_active: i2c_3_active {
				/* active state */
				mux {
					pins = "gpio10", "gpio11";
					function = "blsp_i2c3";
				};

				config {
					pins = "gpio10", "gpio11";
					drive-strength = <2>;
					bias-disable;
				};
			};

			i2c_3_sleep: i2c_3_sleep {
				/* suspended state */
				mux {
					pins = "gpio10", "gpio11";
					function = "gpio";
				};

				config {
					pins = "gpio10", "gpio11";
					drive-strength = <2>;
					bias-disable;
				};
			};
		};

		i2c_5 {
			i2c_5_active: i2c_5_active {
				/* active state */
				mux {
					pins = "gpio18", "gpio19";
					function = "blsp_i2c5";
				};

				config {
					pins = "gpio18", "gpio19";
					drive-strength = <2>;
					bias-disable;
				};
			};

			i2c_5_sleep: i2c_5_sleep {
				/* suspended state */
				mux {
					pins = "gpio18", "gpio19";
					function = "gpio";
				};

				config {
					pins = "gpio18", "gpio19";
					drive-strength = <2>;
					bias-disable;
				};
			};
		};
        
        i2c_6 {
			i2c_6_active: i2c_6_active {
				/* active state */
				mux {
					pins = "gpio22", "gpio23";
					function = "blsp_i2c6";
				};

				config {
					pins = "gpio22", "gpio23";
					drive-strength = <2>;
					bias-disable;
				};
			};

			i2c_6_sleep: i2c_6_sleep {
				/* suspended state */
				mux {
					pins = "gpio22", "gpio23";
					function = "gpio";
				};

				config {
					pins = "gpio22", "gpio23";
					drive-strength = <2>;
					output-high;
				};
			};
		};
```	
	
#### I2C参数配置
msm8953.dtsi
```
&soc {
    #address-cells = <1>; 
    #size-cells = <1>; 
    ranges = <0 0 0 0xffffffff>;
    compatible = "simple-bus";
	[...省略...]
	
    i2c_2: i2c@78b6000 { /* BLSP1 QUP2 */
        compatible = "qcom,i2c-msm-v2";
        #address-cells = <1>; 
        #size-cells = <0>; 
        reg-names = "qup_phys_addr";
        reg = <0x78b6000 0x600>;		// i2c寄存器对应的物理地址和长度
        interrupt-names = "qup_irq";
        #size-cells = <0>;
        reg-names = "qup_phys_addr";
        reg = <0x78b6000 0x600>;
        interrupt-names = "qup_irq";
        interrupts = <0 96 0>;			// 中断号96
        qcom,clk-freq-out = <400000>;	// 输出时钟
        qcom,clk-freq-in  = <19200000>;	// 内部时钟
        clock-names = "iface_clk", "core_clk";
        clocks = <&clock_gcc clk_gcc_blsp1_ahb_clk>,
            <&clock_gcc clk_gcc_blsp1_qup2_i2c_apps_clk>;

        pinctrl-names = "i2c_active", "i2c_sleep";		// pinctrl配置，参考msm8953-pinctrl.dtsi
        pinctrl-0 = <&i2c_2_active>;
        pinctrl-1 = <&i2c_2_sleep>;
        qcom,noise-rjct-scl = <0>; 
        qcom,noise-rjct-sda = <0>; 
        qcom,master-id = <86>;
        dmas = <&dma_blsp1 6 64 0x20000020 0x20>,
            <&dma_blsp1 7 32 0x20000020 0x20>;
        dma-names = "tx", "rx";

        /* DSI_TO_HDMI I2C configuration */
        adv7533@39 {
            compatible = "adv7533";
            reg = <0x39>;
            instance_id = <0>; 
            adi,video-mode = <3>; /* 3 = 1080p */
            adi,main-addr = <0x39>;
            adi,cec-dsi-addr = <0x3C>;
            adi,enable-audio;
            pinctrl-names = "pmx_adv7533_active",
                        "pmx_adv7533_suspend";
            pinctrl-0 = <&adv7533_int_active>;
            pinctrl-1 = <&adv7533_int_suspend>;
            adi,irq-gpio = <&tlmm 90 0x2002>;
            hpd-5v-en-supply = <&adv_vreg>;
            qcom,supply-names = "hpd-5v-en";
            qcom,min-voltage-level = <0>;
            qcom,max-voltage-level = <0>;
            qcom,enable-load = <0>;
            qcom,disable-load = <0>;
        };
    };
	...
	    i2c_6: i2c@7af6000 { /* BLSP2 QUP2 */
        compatible = "qcom,i2c-msm-v2";
        #address-cells = <1>;
        #size-cells = <0>;
        reg-names = "qup_phys_addr";
        reg = <0x7af6000 0x600>;
        interrupt-names = "qup_irq";
        interrupts = <0 300 0>;
        qcom,clk-freq-out = <400000>;
        qcom,clk-freq-in  = <19200000>;
        clock-names = "iface_clk", "core_clk";
        clocks = <&clock_gcc clk_gcc_blsp2_ahb_clk>,
            <&clock_gcc clk_gcc_blsp2_qup2_i2c_apps_clk>;

        pinctrl-names = "i2c_active", "i2c_sleep";
        pinctrl-0 = <&i2c_6_active>;
        pinctrl-1 = <&i2c_6_sleep>;
        qcom,noise-rjct-scl = <0>;
        qcom,noise-rjct-sda = <0>;
        qcom,master-id = <84>;
        dmas = <&dma_blsp2 6 64 0x20000020 0x20>,
            <&dma_blsp2 7 32 0x20000020 0x20>;
        dma-names = "tx", "rx";
        at24@50 {
                compatible = "atmel,24c256";
                reg = <0x50>;
                pagesize = <64>;
        };
    };
	...
	
	给i2c起别名：
	    aliases {
        /* smdtty devices */
        smd1 = &smdtty_apps_fm;
        smd2 = &smdtty_apps_riva_bt_acl;
        smd3 = &smdtty_apps_riva_bt_cmd;
        smd4 = &smdtty_mbalbridge;
        smd5 = &smdtty_apps_riva_ant_cmd;
        smd6 = &smdtty_apps_riva_ant_data;
        smd7 = &smdtty_data1;
        smd8 = &smdtty_data4;
        smd11 = &smdtty_data11;
        smd21 = &smdtty_data21;
        smd36 = &smdtty_loopback;
        sdhc1 = &sdhc_1; /* SDC1 eMMC slot */
        sdhc2 = &sdhc_2; /* SDC2 for SD card */
        i2c2 = &i2c_2;
        i2c3 = &i2c_3;
        i2c5 = &i2c_5;
        i2c6 = &i2c_6;
        i2c8 = &i2c_8;
        /* spi3 = &spi_3; */
    };
```
	
#### 添加i2c设备
arch/arm/boot/dts/IDH60/msm8953-mtp.dtsi
```
	&i2c_2 {
       bq@55{
               compatible = "bq,bq27541";
               reg = <0x55>;		// i2设备地址
       };  
       tsu6721@25 {
        reg = <0x25>;
        compatible = "ti,tsu6721";
        tsu6721,irq-gpio = <&tlmm 97 0x02>; /* IRQF_TRIGGER_FALLING */
    };  

&i2c_5 { /* BLSP2 QUP1 (NFC) */
    nq@28 {
        compatible = "qcom,nq-nci";
        reg = <0x28>;
        qcom,nq-irq = <&tlmm 36 0x00>;
        qcom,nq-ven = <&tlmm 38 0x00>;
        qcom,nq-firm = <&tlmm 62 0x00>;
        qcom,nq-clkreq = <&pm8953_gpios 2 0x00>;
        interrupt-parent = <&tlmm>;
        qcom,clk-src = "BBCLK2";
        /*interrupts = <36 0>;*/
        interrupt-names = "nfc_irq";
        pinctrl-names = "nfc_active", "nfc_suspend";
        pinctrl-0 = <&nfc_int_active >;
        pinctrl-1 = <&nfc_int_suspend>;
        clocks = <&clock_gcc clk_bb_clk2_pin>;
        clock-names = "ref_clk";
        chip-type = <0x41>;
		};
    pn548@28 {
                compatible = "NXP,pn548";
                reg = <0x28>;
                vdd-supply = <&pm8953_l10>;
                vcc_i2c-supply = <&pm8953_l5>;
                interrupt-parent = <&tlmm>;
                /*interrupts = <36 0x0>;*/
                irq-gpio = <&tlmm 36 0x00>;
                ven-gpio = <&tlmm 38 0x00>;
                firm-gpio = <&tlmm 62 0x00>;
        chip-type = <0x28>;
       };
};
```