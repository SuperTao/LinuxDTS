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

        pmx-uartconsole {
            uart_console_active: uart_console_active {	// active
                mux {
                    pins = "gpio4", "gpio5";		// GPIO引脚
                    function = "blsp_uart2";		// 复用成uart
                };   

                config {
                    pins = "gpio4", "gpio5";
                    drive-strength = <2>; 	// 电流2MA
                    bias-disable;			// 无上拉下拉
                };   
            };   

            uart_console_sleep: uart_console_sleep {	// sleep
                mux {
                    pins = "gpio4", "gpio5";
                    function = "blsp_uart2";	// uart
                };   

                config {
                    pins = "gpio4", "gpio5";
                    drive-strength = <2>; 	// 2MA 
                    bias-pull-down;			// 下拉
                };   
            };   

        };   
```
#### UART参数配置
msm8953.dtsi
```
    blsp1_uart0: serial@78af000 {
        compatible = "qcom,msm-lsuart-v14";
        reg = <0x78af000 0x200>;
        interrupts = <0 107 0>;
        status = "disabled";
        clocks = <&clock_gcc clk_gcc_blsp1_uart1_apps_clk>,
        <&clock_gcc clk_gcc_blsp1_ahb_clk>;
        clock-names = "core_clk", "iface_clk";
    };

    blsp1_uart1: uart@78b0000 {
        compatible = "qcom,msm-hsuart-v14";
        reg = <0x78b0000 0x200>,
            <0x7884000 0x1f000>;
        reg-names = "core_mem", "bam_mem";
        
        interrupt-names = "core_irq", "bam_irq", "wakeup_irq";
        #address-cells = <0>;
        interrupt-parent = <&blsp1_uart1>;
        interrupts = <0 1 2>;
        #interrupt-cells = <1>;
        interrupt-map-mask = <0xffffffff>;
        interrupt-map = <0 &intc 0 108 0
                1 &intc 0 238 0
                2 &tlmm 13 0>;

        qcom,inject-rx-on-wakeup;
        qcom,rx-char-to-inject = <0xFD>;
        qcom,master-id = <86>;
        clock-names = "core_clk", "iface_clk";
        clocks = <&clock_gcc clk_gcc_blsp1_uart2_apps_clk>,
            <&clock_gcc clk_gcc_blsp1_ahb_clk>;
        pinctrl-names = "sleep", "default";
        pinctrl-0 = <&hsuart_sleep>;
        pinctrl-1 = <&hsuart_active>;
        qcom,bam-tx-ep-pipe-index = <2>;
        qcom,bam-rx-ep-pipe-index = <3>;
        qcom,msm-bus,name = "blsp1_uart1";
        qcom,msm-bus,num-cases = <2>;
        qcom,msm-bus,num-paths = <1>;
        qcom,msm-bus,vectors-KBps =
                <86 512 0 0>,
				                <86 512 500 800>;
        status = "disabled";
    };

    blsp2_uart0: uart@7aef000 {
        compatible = "qcom,msm-hsuart-v14";
        reg = <0x7aef000 0x200>,
            <0x7ac4000 0x1f000>;
        reg-names = "core_mem", "bam_mem";

        interrupt-names = "core_irq", "bam_irq", "wakeup_irq";
        #address-cells = <0>;
        interrupt-parent = <&blsp2_uart0>;
        interrupts = <0 1 2>;
        #interrupt-cells = <1>;
        interrupt-map-mask = <0xffffffff>;
        interrupt-map = <0 &intc 0 306 0
                1 &intc 0 239 0
                2 &tlmm 17 0>;

        qcom,inject-rx-on-wakeup;
        qcom,rx-char-to-inject = <0xFD>;
        qcom,master-id = <84>;
        clock-names = "core_clk", "iface_clk";
        clocks = <&clock_gcc clk_gcc_blsp2_uart1_apps_clk>,
            <&clock_gcc clk_gcc_blsp2_ahb_clk>;
        pinctrl-names = "sleep", "default";
        pinctrl-0 = <&blsp2_uart0_sleep>;
        pinctrl-1 = <&blsp2_uart0_active>;
        qcom,bam-tx-ep-pipe-index = <0>;
        qcom,bam-rx-ep-pipe-index = <1>;
        qcom,msm-bus,name = "blsp2_uart0";
        qcom,msm-bus,num-cases = <2>;
        qcom,msm-bus,num-paths = <1>;
        qcom,msm-bus,vectors-KBps =
                <84 512 0 0>,
                <84 512 500 800>;
        status = "disabled";
    };
```

msm8953-mtp.dtsi
```
	&blsp1_uart0 {
    status = "ok";
    pinctrl-names = "default";
    pinctrl-0 = <&uart_console_active>;
};
```