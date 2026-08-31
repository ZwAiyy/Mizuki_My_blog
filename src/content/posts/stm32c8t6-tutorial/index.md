---
title: "STM32C8T6入门教程：Keil + 标准库开发指南"
published: 2024-12-19
description: "手把手教你使用Keil MDK和标准库开发STM32C8T6，从环境搭建到第一个LED程序。"
tags: ["STM32", "嵌入式", "Keil", "C语言", "单片机"]
category: 硬件开发
draft: false
---

STM32F103C8T6 是 STMicroelectronics 推出的基于 ARM Cortex-M3 内核的 32 位微控制器，因其性价比高、资源丰富，成为嵌入式开发入门的首选芯片。

## 开发环境搭建

### 1. 安装 Keil MDK

1. 访问 [Keil 官网](https://www.keil.com/) 下载 MDK-Arm
2. 安装过程中选择 STM32F1 系列支持包
3. 安装完成后，需要安装 STM32F1xx 设备支持包 (DFP)

### 2. 下载标准库

从 ST 官网下载 STM32F10x 标准外设库，目录结构如下：

```
STM32F10x_StdPeriph_Lib_V3.5.0/
├── Libraries/
│   ├── CMSIS/          # Cortex-M3 内核支持
│   └── STM32F10x_StdPeriph_Driver/  # 标准外设驱动
├── Project/
│   └── Examples/       # 官方示例代码
└── Utilities/
```

## 第一个工程：LED 闪烁

### 1. 创建工程

1. 打开 Keil，选择 `Project → New μVision Project`
2. 选择芯片型号：`STM32F103C8`
3. 添加标准库文件到工程

### 2. 配置头文件

在 `stm32f10x_conf.h` 中启用需要的外设：

```c
#include "stm32f10x_gpio.h"
#include "stm32f10x_rcc.h"
```

### 3. 编写代码

```c
#include "stm32f10x.h"
#include "stm32f10x_gpio.h"
#include "stm32f10x_rcc.h"

// 延时函数
void Delay(uint32_t nCount)
{
    for(; nCount != 0; nCount--);
}

// GPIO 初始化
void LED_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    
    // 开启 GPIOC 时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
    
    // 配置 PC13 为推挽输出
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);
}

int main(void)
{
    LED_Init();
    
    while(1)
    {
        // 点亮 LED (低电平有效)
        GPIO_ResetBits(GPIOC, GPIO_Pin_13);
        Delay(5000000);
        
        // 熄灭 LED
        GPIO_SetBits(GPIOC, GPIO_Pin_13);
        Delay(5000000);
    }
}
```

## 常用外设配置

### USART 串口配置

```c
void USART1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;
    
    // 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA, ENABLE);
    
    // 配置 TX (PA9) 为复用推挽输出
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
    
    // 配置 RX (PA10) 为浮空输入
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
    
    // 配置串口参数
    USART_InitStructure.USART_BaudRate = 115200;
    USART_InitStructure.USART_WordLength = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits = USART_StopBits_1;
    USART_InitStructure.USART_Parity = USART_Parity_No;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
    USART_Init(USART1, &USART_InitStructure);
    
    // 使能串口
    USART_Cmd(USART1, ENABLE);
}
```

## 调试与烧录

### ST-Link 配置

1. 连接 ST-Link 到开发板的 SWD 接口
2. 在 Keil 中选择 `Options for Target → Debug`
3. 选择 `ST-Link Debugger`
4. 点击 `Settings` 确认识别到芯片

### 程序下载

1. 编译工程：`Build` 按钮或 `F7`
2. 下载程序：`Download` 按钮或 `F8`
3. 复位运行：`Reset` 按钮

## 常见问题

### 1. 编译错误：未定义符号

确保已添加所有必要的标准库文件，并在 `stm32f10x_conf.h` 中启用了对应的头文件。

### 2. 下载失败

- 检查 ST-Link 连接
- 确认芯片供电正常
- 尝试降低 SWD 时钟频率

### 3. 程序不运行

- 检查启动文件是否正确
- 确认时钟配置
- 检查复位电路

## 学习资源

- [STM32F103 数据手册](https://www.st.com/resource/en/datasheet/stm32f103c8.pdf)
- [STM32 标准库用户手册](https://www.st.com/resource/en/user_manual/um0427-stm32f10xxx-12xxx-13xxx-20xxx-21xxx-23xxx-standard-peripheral-library-stmicroelectronics.pdf)
- [Keil MDK 文档](https://www.keil.com/support/docs/)

## 总结

STM32C8T6 是嵌入式入门的理想选择，通过 Keil 和标准库可以快速上手。掌握 GPIO、USART 等基础外设后，可以进一步学习定时器、ADC、I2C、SPI 等高级功能。

下一步可以尝试：
- 使用中断替代延时函数
- 配置定时器实现精确延时
- 学习 I2C/SPI 通信协议
- 尝试 HAL 库开发方式