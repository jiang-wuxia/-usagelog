# 富芮坤—fr30xx系列-低功耗—PMU-使用记录
## PMU在睡眠模式中的作用

### 核心特点
- **芯片进入睡眠模式后，只有PMU能正常工作**。

### 外设状态
- **掉电且无法运行**的外设：
  - GPIO（除特定引脚外）
  - ADC
  - UART
  - I2C
  - SPI
  - PWM
  - DAC

### 如何重新启用外设
1. **设置唤醒脚** 来唤醒芯片。
2. **芯片唤醒后**，外设才能恢复正常使用。

## 睡眠模式
---

工作模式：`system_prevent_sleep_set` 通过这个函数设置对应的标志位，芯片就无法进入睡眠模式。  
睡眠模式：`system_prevent_sleep_clear` 这个函数可以让芯片进入睡眠模式。  

---
正常使用就是：使用`system_prevent_sleep_set`设置对应的标志位，使芯片无法进入睡眠模式，然后使用执行需要做的操作，最后使用`system_prevent_sleep_clear`使芯片进入睡眠模式。
芯片进入睡眠模式后，只有PMU能正常工作，其他的外设都无法正常工作。芯片的IO口态可以通过 进入睡眠模式时的中断函数来设置  


芯片退出睡眠模式后，需在相应的函数里重新初始化使用的外设。

标志位：
```c
#define SYSTEM_PREVENT_SLEEP_TYPE_DISABLE       0x00000001
#define SYSTEM_PREVENT_SLEEP_TYPE_CALIBRATION   0x00000002
#define SYSTEM_PREVENT_SLEEP_TYPE_DSP           0x00000004
#define SYSTEM_PREVENT_SLEEP_TYPE_HCI_RX        0x00000008
#define SYSTEM_PREVENT_SLEEP_TYPE_HCI_TX        0x00000010
#define SYSTEM_PREVENT_SLEEP_SCO_ONGOING        0x00000020
#define SYSTEM_PREVENT_SLEEP_A2DP_ONGOING       0x00000040
#define SYSTEM_PREVENT_SLEEP_DISPLAY_ONGOING    0x00000080
```  
### 睡眠前进入的函数：`user_entry_before_sleep`  
```c
__RAM_CODE void user_entry_before_sleep(void)
{
    while(!(Uart3_handle.UARTx->USR.TFE));
    system_delay_us(100);
    ool_write16(PMU_REG_PIN_PULL_EN, 0x3fff);
    ool_write16(PMU_REG_PIN_PULL_SEL, 0x3ffd);

    ool_write(PMU_REG_PMU_GATE_M, ool_read(PMU_REG_PMU_GATE_M) | 0x40);
}

```  

### 退出睡眠模式后执行的函数：`user_entry_after_sleep`  
```c
__RAM_CODE void user_entry_after_sleep(void)
{
    /* 
     * enable pull up of all 3.3v IO, these configuration will be latched by set 
     * BIT6 of PMU_REG_PMU_GATE_M regsiter. used to avoid electric leakage
     */
    SYSTEM->PortA_PullSelect = 0x0000ffff;
    SYSTEM->PortB_PullSelect = 0x0000fff7;
    SYSTEM->PortC_PullSelect = 0x0000ffff;
    SYSTEM->PortD_PullSelect = 0x0000ffff;
    SYSTEM->PortE_PullSelect = 0x0000ffff;
    SYSTEM->PortA_PullEN = 0x0000ffff;
    SYSTEM->PortB_PullEN = 0x0000ffff;
    SYSTEM->PortC_PullEN = 0x0000ffff;
    SYSTEM->PortD_PullEN = 0x0000ffff;
    SYSTEM->PortE_PullEN = 0x0000ffff;
    SYSTEM->QspiPadConfig.QSPI_PullEN = 0x0000000;
    
    SYSTEM->PortA_InputOpenCircuit = 0xFFFFFFFF;
    SYSTEM->PortB_InputOpenCircuit = 0xFFFFFFFF;
    SYSTEM->PortC_InputOpenCircuit = 0xFFFFFFFF;
    SYSTEM->PortD_InputOpenCircuit = 0xFFFFFFFF;
    SYSTEM->PortE_InputOpenCircuit = 0xFFFFFFFF;

    hw_clock_init();
    hw_xip_flash_init(true);
    host_hci_reinit();
    ool_write(PMU_REG_PMU_GATE_M, ool_read(PMU_REG_PMU_GATE_M) & (~0x40));
    
    GPIO_InitTypeDef gpio_config;
    
    /* configure all interrupt priority to 2 */
    *(volatile uint32_t *)0xE000E400 = 0x40404040;
    *(volatile uint32_t *)0xE000E404 = 0x40404040;
    *(volatile uint32_t *)0xE000E408 = 0x40404040;
    *(volatile uint32_t *)0xE000E40C = 0x40404040;
    *(volatile uint32_t *)0xE000E410 = 0x40404040;
    *(volatile uint32_t *)0xE000E414 = 0x40404040;
    *(volatile uint32_t *)0xE000E418 = 0x40404040;
    *(volatile uint32_t *)0xE000E41C = 0x40404040;
    *(volatile uint32_t *)0xE000E420 = 0x40404040;
    *(volatile uint32_t *)0xE000E424 = 0x40404040;
    *(volatile uint32_t *)0xE000E428 = 0x40404040;
    *(volatile uint32_t *)0xE000E42C = 0x40404040;
    *(volatile uint32_t *)0xE000E430 = 0x40404040;
    *(volatile uint32_t *)0xE000E434 = 0x40404040;
    *(volatile uint32_t *)0xE000E438 = 0x40404040;
    *(volatile uint32_t *)0xE000E43C = 0x40404040;
    *(volatile uint32_t *)0xE000E440 = 0x40404040;

    NVIC_SetPriority(UART0_IRQn, 2);
    NVIC_EnableIRQ(UART0_IRQn);
    NVIC_SetPriority(PMU_IRQn, 4);
    NVIC_EnableIRQ(PMU_IRQn);
    
    /* configure PB4 and PB5 to UART3 function */
    __SYSTEM_GPIOA_CLK_ENABLE();
    gpio_config.Pin = GPIO_PIN_4 | GPIO_PIN_5;
    gpio_config.Mode = GPIO_MODE_AF_PP;
    gpio_config.Pull = GPIO_PULLUP;
    gpio_config.Alternate = GPIO_FUNCTION_1;
    gpio_init(GPIOB, &gpio_config);
    
    /* UART0: used for Log and AT command */
    __SYSTEM_UART3_CLK_ENABLE();
    Uart3_handle.UARTx = UART3;
    Uart3_handle.Init.BaudRate   = 921600;
    Uart3_handle.Init.DataLength = UART_DATA_LENGTH_8BIT;
    Uart3_handle.Init.StopBits   = UART_STOPBITS_1;
    Uart3_handle.Init.Parity     = UART_PARITY_NONE;
    Uart3_handle.Init.FIFO_Mode  = UART_FIFO_ENABLE;
    Uart3_handle.TxCpltCallback  = NULL;
    Uart3_handle.RxCpltCallback  = app_at_rx_done;
    uart_init(&Uart3_handle);
    NVIC_SetPriority(UART3_IRQn, 4);
    NVIC_EnableIRQ(UART3_IRQn);
    
    {
        static bool first_wakeup = true;
        bool do_calib = false;
        static TickType_t last_tick;
        TickType_t curr_tick;
        if (first_wakeup) {
            first_wakeup = false;
            last_tick = xTaskGetTickCount();
            curr_tick = last_tick;
            do_calib = true;
        }
        else {
            curr_tick = xTaskGetTickCount();
            if ((curr_tick - last_tick) > 10000) {
                last_tick = curr_tick;
                do_calib = true;
            }
        }
        
        if (do_calib) {
            /* restart calibration */
            __SYSTEM_CALI_CLK_ENABLE();
            cali_handle.mode = CALI_UP_MODE_NORMAL;
            cali_handle.rc_cnt = 60;
            cali_handle.DoneCallback = cali_done_handle;
            cali_init(&cali_handle);
            cali_start_IT(&cali_handle);
            system_prevent_sleep_set(SYSTEM_PREVENT_SLEEP_TYPE_CALIBRATION);
            NVIC_SetPriority(CALI_IRQn, 2);
            NVIC_EnableIRQ(CALI_IRQn);
        }
    }
    
    gpio_wakeup_func_set(GPIOD, GPIO_PIN_0, current_PD0_level);
}
```

## 睡眠模式下的IO配置

1.睡眠模式下保持配置的IO口电平（普通IO口）

   1.在主函数进行初始化相应的模式
   2.在唤醒时进入的函数重新初始化：**user_entry_after_sleep_user** 或者 **user_entry_after_sleep**
   3.要改变电平要在系统唤醒的时候操作









