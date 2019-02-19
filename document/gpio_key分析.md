## 设备树GPIO初始化
#### 配置引脚的复用
msm8953-pinctrl.dtsi
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
		tlmm_gpio_key {
            gpio_key_active: gpio_key_active {				// activ状态
                mux {
                    pins = "gpio85", "gpio9", "gpio90";		// gpio引脚序号
                    function = "gpio";						// gpio引脚,设置成gpio功能
                };   

                config {
                    pins = "gpio85", "gpio9", "gpio90";
                    drive-strength = <2>; 		// 电流强度
                    bias-pull-up;				// 上拉
                };   
            };   

            gpio_key_suspend: gpio_key_suspend {			// suspend状态
                mux {
                    pins = "gpio85", "gpio9", "gpio90";
                    function = "gpio";
                };   

                config {
                    pins = "gpio85", "gpio9", "gpio90";
                    drive-strength = <2>; 
                    bias-pull-up;
                };   
            };   
        };
```
#### 配置gpio的参数
msm8953-mtp.dtsi
```
		gpio_keys {
        compatible = "gpio-keys";
        input-name = "gpio-keys";
        virtual-key-code = <143>;
        pinctrl-names = "tlmm_gpio_key_active","tlmm_gpio_key_suspend";
        pinctrl-0 = <&gpio_key_active>;		// active状态pinctrl
        pinctrl-1 = <&gpio_key_suspend>;	// suspend状态pinctrl

        vol_up {
            label = "volume_up";
            gpios = <&tlmm 85 0x1>;		// tlmm， 第85个引脚，电平
            linux,input-type = <1>;		// 设置成输入
            linux,code = <115>;			// 键值115
            debounce-interval = <15>;	// 去抖动时间
            gpio-key,wakeup;
        };  

        scan_right {
            label = "scanner_right";
            gpios = <&tlmm 90 0x1>;
            linux,input-type = <1>;
            linux,code = <261>;
            debounce-interval = <15>;
            gpio-key,wakeup;
        };  

        scan_left {
            label = "scanner_left";
            gpios = <&tlmm 9 0x1>;
            linux,input-type = <1>;
            linux,code = <257>;
            debounce-interval = <15>;
            gpio-key,wakeup;
        };  
    };
```
## 驱动注册
#### 主要内容
* 实现设备树节点的解读
* 设置gpio状态

drivers/input/keyboard/gpio_keys.c
```
static int __init gpio_keys_init(void)
{
​    return platform_driver_register(&gpio_keys_device_driver);
}	

static struct platform_driver gpio_keys_device_driver = {
​    .probe      = gpio_keys_probe,
​    .remove     = gpio_keys_remove,
​    .driver     = {
​        .name   = "gpio-keys",
​        .owner  = THIS_MODULE,
​        .pm = &gpio_keys_pm_ops,
​        .of_match_table = of_match_ptr(gpio_keys_of_match),		// 匹配表
​    }
};
```
of_match_ptr中的字符串需要与设备树中的名称相同才能匹配成功
```
static const struct of_device_id gpio_keys_of_match[] = {
​    { .compatible = "gpio-keys", },
​    { }, 
};
MODULE_DEVICE_TABLE(of, gpio_keys_of_match);
​```
匹配成功之后，调用probe函数
```
static int gpio_keys_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	const struct gpio_keys_platform_data *pdata = dev_get_platdata(dev);
	struct gpio_keys_drvdata *ddata;
	struct input_dev *input;
	size_t size;
	int i, error;
	int wakeup = 0;
	struct pinctrl_state *set_state;

	if (!pdata) {
		// 读取设备树中的节点信息
		pdata = gpio_keys_get_devtree_pdata(dev);
		if (IS_ERR(pdata))
			return PTR_ERR(pdata);
	}

	size = sizeof(struct gpio_keys_drvdata) +
			pdata->nbuttons * sizeof(struct gpio_button_data);
	ddata = devm_kzalloc(dev, size, GFP_KERNEL);
	if (!ddata) {
		dev_err(dev, "failed to allocate state\n");
		return -ENOMEM;
	}

	input = devm_input_allocate_device(dev);
	if (!input) {
		dev_err(dev, "failed to allocate input device\n");
		return -ENOMEM;
	}

	global_dev = dev;
	ddata->pdata = pdata;
	ddata->input = input;
	mutex_init(&ddata->disable_lock);

	platform_set_drvdata(pdev, ddata);
	input_set_drvdata(input, ddata);

	input->name = GPIO_KEYS_DEV_NAME;
	input->phys = "gpio-keys/input0";
	input->dev.parent = &pdev->dev;
	input->open = gpio_keys_open;	// 打开函数
	input->close = gpio_keys_close;	// 关闭函数

	input->id.bustype = BUS_HOST;
	input->id.vendor = 0x0001;
	input->id.product = 0x0001;
	input->id.version = 0x0100;

	/* Enable auto repeat feature of Linux input subsystem */
	if (pdata->rep)
		__set_bit(EV_REP, input->evbit);

	/* Get pinctrl if target uses pinctrl */
	ddata->key_pinctrl = devm_pinctrl_get(dev);
	if (IS_ERR(ddata->key_pinctrl)) {
		if (PTR_ERR(ddata->key_pinctrl) == -EPROBE_DEFER)
			return -EPROBE_DEFER;

		pr_debug("Target does not use pinctrl\n");
		ddata->key_pinctrl = NULL;
	}

	if (ddata->key_pinctrl) {
		error = gpio_keys_pinctrl_configure(ddata, true);
		if (error) {
			dev_err(dev, "cannot set ts pinctrl active state\n");
			return error;
		}
	}

	for (i = 0; i < pdata->nbuttons; i++) {
		const struct gpio_keys_button *button = &pdata->buttons[i];
		struct gpio_button_data *bdata = &ddata->data[i];
		// 根据设备数设置gpio的状态
		error = gpio_keys_setup_key(pdev, input, bdata, button);
		if (error)
			goto err_setup_key;

		if (button->wakeup)
			wakeup = 1;
	}

#ifdef CONFIG_HSM_KEYS_WAKEUP
	if (pdata->nbuttons) {
		trans_key_array_size = (unsigned char) pdata->nbuttons;
		transformed_key_code = (unsigned int *)
			kzalloc (trans_key_array_size * sizeof(unsigned int),
					GFP_KERNEL);
	}
        if (virtual_key_code)
                input_set_capability(input, EV_KEY, virtual_key_code);
#endif
	error = sysfs_create_group(&pdev->dev.kobj, &gpio_keys_attr_group);
	if (error) {
		dev_err(dev, "Unable to export keys/switches, error: %d\n",
			error);
		goto err_create_sysfs;
	}
	// 输入设备注册
	error = input_register_device(input);
	if (error) {
		dev_err(dev, "Unable to register input device, error: %d\n",
			error);
		goto err_remove_group;
	}

	device_init_wakeup(&pdev->dev, wakeup);

	if (pdata->use_syscore)
		gpio_keys_syscore_pm_ops.resume = gpio_keys_syscore_resume;

	register_syscore_ops(&gpio_keys_syscore_pm_ops);

	return 0;

err_remove_group:
	sysfs_remove_group(&pdev->dev.kobj, &gpio_keys_attr_group);
err_create_sysfs:
err_setup_key:
	if (ddata->key_pinctrl) {
		set_state =
		pinctrl_lookup_state(ddata->key_pinctrl,
						"tlmm_gpio_key_suspend");
		if (IS_ERR(set_state))
			dev_err(dev, "cannot get gpiokey pinctrl sleep state\n");
		else
			pinctrl_select_state(ddata->key_pinctrl, set_state);
	}

	return error;
}
```
#### 设备树参数解读
```
/*
 * Translate OpenFirmware node properties into platform_data
 */
static struct gpio_keys_platform_data *
gpio_keys_get_devtree_pdata(struct device *dev)
{
	struct device_node *node, *pp;
	struct gpio_keys_platform_data *pdata;
	struct gpio_keys_button *button;
	int error;
	int nbuttons;
	int i;

	node = dev->of_node;
	if (!node)
		return ERR_PTR(-ENODEV);

	nbuttons = of_get_child_count(node);
	if (nbuttons == 0)
		return ERR_PTR(-ENODEV);

	pdata = devm_kzalloc(dev,
			     sizeof(*pdata) + nbuttons * sizeof(*button),
			     GFP_KERNEL);
	if (!pdata)
		return ERR_PTR(-ENOMEM);

	pdata->buttons = (struct gpio_keys_button *)(pdata + 1);
	pdata->nbuttons = nbuttons;
	
	pdata->rep = !!of_get_property(node, "autorepeat", NULL);
	// 读取设备数节点中的属性, 没有就给NULL
	pdata->name = of_get_property(node, "input-name", NULL);
	pdata->use_syscore = of_property_read_bool(node, "use-syscore");
#ifdef CONFIG_HSM_KEYS_WAKEUP
	if (of_property_read_u32(node, "virtual-key-code", &virtual_key_code)) {
			dev_warn(dev, "virtual-key-code for wakeup does not exist\n");
	}
#endif

	i = 0;
	//循环读取设备树节点"gpios"属性，比如gpios = <&tlmm 85 0x1>;
	for_each_child_of_node(node, pp) {
		int gpio;
		enum of_gpio_flags flags;

		if (!of_find_property(pp, "gpios", NULL)) {
			pdata->nbuttons--;
			dev_warn(dev, "Found button without gpios\n");
			continue;
		}
		// 读取gpio的序号
		gpio = of_get_gpio_flags(pp, 0, &flags);
		if (gpio < 0) {
			error = gpio;
			if (error != -EPROBE_DEFER)
				dev_err(dev,
					"Failed to get gpio flags, error: %d\n",
					error);
			return ERR_PTR(error);
		}

		button = &pdata->buttons[i++];

		button->gpio = gpio;
		// 查看gpio的电平
		button->active_low = flags & OF_GPIO_ACTIVE_LOW;
		// 读取设备数的linux,code属性，例如linux,code=<115>
		// 这个对应的应该是gpio对应的具体键盘的键值
		if (of_property_read_u32(pp, "linux,code", &button->code)) {
			dev_err(dev, "Button without keycode: 0x%x\n",
				button->gpio);
			return ERR_PTR(-EINVAL);
		}
		// 读取label属性, label = "volume up"
		button->desc = of_get_property(pp, "label", NULL);
		// 读取linux,input-type, linux,input-type = <1>
		if (of_property_read_u32(pp, "linux,input-type", &button->type))
			button->type = EV_KEY;
		// gpio-key,wakeup
		button->wakeup = !!of_get_property(pp, "gpio-key,wakeup", NULL);
		// debounce-interval = <15>, 按键去抖
		if (of_property_read_u32(pp, "debounce-interval",
					&button->debounce_interval))
			button->debounce_interval = 5;
	}

	if (pdata->nbuttons == 0)
		return ERR_PTR(-EINVAL);

	return pdata;
}
```
#### gpio按键设置
```
static int gpio_keys_setup_key(struct platform_device *pdev,
				struct input_dev *input,
				struct gpio_button_data *bdata,
				const struct gpio_keys_button *button)
{
	const char *desc = button->desc ? button->desc : "gpio_keys";
	struct device *dev = &pdev->dev;
	irq_handler_t isr;
	unsigned long irqflags;
	int irq;
	int error;

	bdata->input = input;
	bdata->button = button;
	spin_lock_init(&bdata->lock);

	if (gpio_is_valid(button->gpio)) {
		// 申请GPIO，设置成输入
		error = devm_gpio_request_one(&pdev->dev, button->gpio,
					      GPIOF_IN, desc);
		if (error < 0) {
			dev_err(dev, "Failed to request GPIO %d, error %d\n",
				button->gpio, error);
			return error;
		}
		// 设置按键去抖动
		if (button->debounce_interval) {
			error = gpio_set_debounce(button->gpio,
					button->debounce_interval * 1000);
			/* use timer if gpiolib doesn't provide debounce */
			if (error < 0)
				bdata->timer_debounce =
						button->debounce_interval;
		}
		// 申请gpio中断
		irq = gpio_to_irq(button->gpio);
		if (irq < 0) {
			error = irq;
			dev_err(dev,
				"Unable to get irq number for GPIO %d, error %d\n",
				button->gpio, error);
			return error;
		}
		bdata->irq = irq;

		INIT_WORK(&bdata->work, gpio_keys_gpio_work_func);
		setup_timer(&bdata->timer,
			    gpio_keys_gpio_timer, (unsigned long)bdata);

		isr = gpio_keys_gpio_isr;
		irqflags = IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING;

	} else {
		if (!button->irq) {
			dev_err(dev, "No IRQ specified\n");
			return -EINVAL;
		}
		bdata->irq = button->irq;

		if (button->type && button->type != EV_KEY) {
			dev_err(dev, "Only EV_KEY allowed for IRQ buttons.\n");
			return -EINVAL;
		}

		bdata->timer_debounce = button->debounce_interval;
		setup_timer(&bdata->timer,
			    gpio_keys_irq_timer, (unsigned long)bdata);

		isr = gpio_keys_irq_isr;
		irqflags = 0;
	}

	input_set_capability(input, button->type ?: EV_KEY, button->code);

	/*
	 * Install custom action to cancel debounce timer and
	 * workqueue item.
	 */
	error = devm_add_action(&pdev->dev, gpio_keys_quiesce_key, bdata);
	if (error) {
		dev_err(&pdev->dev,
			"failed to register quiesce action, error: %d\n",
			error);
		return error;
	}

	/*
	 * If platform has specified that the button can be disabled,
	 * we don't want it to share the interrupt line.
	 */
	if (!button->can_disable)
		irqflags |= IRQF_SHARED;

	error = devm_request_any_context_irq(&pdev->dev, bdata->irq,
					     isr, irqflags, desc, bdata);
	if (error < 0) {
		dev_err(dev, "Unable to claim irq %d; error %d\n",
			bdata->irq, error);
		return error;
	}

	return 0;
}
```