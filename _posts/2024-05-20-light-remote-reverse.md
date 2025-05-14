---
layout: post
title: '逆向工程：将淘宝遥控灯具接入HomeAssistant全记录'
categories: [IoT, Reverse Engineering, CH341A, ASK]
---

## 背景需求
在北京一楼的出租屋里，我通过淘宝购置了一套智能灯具套件。其核心功能是通过遥控器触发舵机按压实体开关，但原始设计仅支持手动遥控操作。为实现**天亮自动开灯**的智能场景，我决定将设备接入HomeAssistant系统。

![遥控器拆解图](https://via.placeholder.com/600x400?text=PCB+with+MCU+and+Antenna)

## 逆向工程挑战
拆解遥控器后发现：
- 主控芯片无任何丝印标识
- 仅配备13.56MHz晶振和PCB天线
- 无编程接口或调试触点
- 通讯协议完全未知

## 协议逆向分析

### 射频定位阶段
1. **频谱扫描**
   使用RTLSDR配合SDR#扫描433MHz频段：
   ```bash
   rtl_sdr -f 433920000 -s 2400000 -n 4800000 capture.iq
   ```
   观察到稳定出现的单峰信号，带宽约10kHz，符合ASK调制特征。

2. **信号捕获**
   AM解调后得到周期性脉冲序列，保存为WAV格式。

### 波形解码阶段
在PulseView中观察到典型编码特征：
- 所有码元均为`001`或`110`组合
- 码元周期严格保持120μs

![信号波形示意图](assets/images/2024-05-20-light-remote/waveform.png)

**关键发现**：
- 采用3位Manchester编码变种
- 逻辑0 → 001（高-低-低）
- 逻辑1 → 110（低-高-高）
- 波特率 = 1s/(120μs/3) = 2500bps

## 硬件实现方案

### 废案

- 树莓派GPIO直出：Linux内核调度抖动导致±50μs误差，无法稳定解码

- 树莓派硬件SPI：片上硬件SPI只能2分频，拆不出2500bps，且SPI tx fifo只有16个字节，用更高的速率去模拟缓冲不够大。

### 最终方案方案（CH341A USB转SPI）
选择CH341A芯片的核心优势：
- 支持任意整数分频：`波特率 = 12MHz / N / M`，N为2幂倍分频，M为整数除法分频
- 4KB发送缓冲

**关键参数计算**：
```
期望波特率 = 2500bps
分频系数 = 12MHz/32/150
```
硬件连接示意图：
```
CH341A          ASK发射模块
MOSI  →────────── DATA
GND   →────────── GND
VCC   →────────── VCC
        
```

### 自动化集成
最终Python控制脚本：
```python
#!/usr/bin/env python3
import usb.core
import usb.util
import sys
import argparse

# CH341A设备参数
VENDOR_ID = 0x1a86
PRODUCT_ID = 0x5512
SPI_CMD = 0xA1
CONFIG_CMD = 0x9A

# 分频设置 (12MHz晶振 -> 32分频 -> 150分频 -> 2500Hz)
DIV_SEQ = [32, 150]  # 分频序列

def configure_spi(dev):
    # 配置SPI模式
    dev.ctrl_transfer(
        bmRequestType=0x40,
        bRequest=CONFIG_CMD,
        wValue=0x2518,  # SPI模式设置
        wIndex=0x00C3,  # 使能SPI
        data_or_wLength=None
    )

    # 设置分频器
    for div in DIV_SEQ:
        dev.ctrl_transfer(
            bmRequestType=0x40,
            bRequest=CONFIG_CMD,
            wValue=0x0F00 | (div & 0xFF),
            wIndex=(div >> 8),
            data_or_wLength=None
        )

def bits_to_bytes(bit_str):
    """将比特字符串转换为字节数组（MSB优先）"""
    # 填充到8的倍数
    padding = 8 - (len(bit_str) % 8)
    if padding != 8:
        bit_str += '0' * padding
    
    byte_array = []
    for i in range(0, len(bit_str), 8):
        byte_str = bit_str[i:i+8]
        byte_array.append(int(byte_str, 2))
    
    return bytes(byte_array)

def send_spi_data(dev, data):
    """发送SPI数据"""
    max_packet = 32  # CH341A单次传输限制
    for i in range(0, len(data), max_packet):
        chunk = data[i:i+max_packet]
        dev.ctrl_transfer(
            bmRequestType=0x40,
            bRequest=SPI_CMD,
            wValue=0x0000,
            wIndex=0x0000,
            data_or_wLength=chunk
        )

def main():
    parser = argparse.ArgumentParser(description="CH341A SPI控制程序")
    parser.add_argument("bits", help="需要发送的比特序列（如 10101100）")
    args = parser.parse_args()

    # 验证输入
    if not all(c in '01' for c in args.bits):
        print("错误：输入必须为二进制字符串（仅包含0和1）")
        sys.exit(1)

    # 查找设备
    dev = usb.core.find(idVendor=VENDOR_ID, idProduct=PRODUCT_ID)
    if dev is None:
        print("未找到CH341A设备！")
        sys.exit(1)

    try:
        # 配置设备
        if dev.is_kernel_driver_active(0):
            dev.detach_kernel_driver(0)
        dev.set_configuration()
        
        configure_spi(dev)
        
        # 转换数据
        byte_data = bits_to_bytes(args.bits)
        print(f"转换后的字节数据: {byte_data.hex()}")
        
        # 发送数据
        send_spi_data(dev, byte_data)
        print("数据发送成功！")

    except usb.core.USBError as e:
        print(f"USB错误: {str(e)}")
        sys.exit(1)
    finally:
        usb.util.dispose_resources(dev)

if __name__ == "__main__":
    main()
```

## 系统集成效果
通过HomeAssistant的Shell Command组件封装控制指令，最终实现自动化场景：
```yaml
automation:
  - alias: "Daylight Auto Lighting"
    trigger:
      platform: numeric_state
      entity_id: sensor.lux_value
      above: 2000
    action:
      service: shell_command.toggle_light
```
