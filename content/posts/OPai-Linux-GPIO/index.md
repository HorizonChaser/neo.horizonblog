+++
title = '嵌入式 Linux 外设 [Part 1] -- GPIO 与 I2C'
date = 2024-01-11T11:49:18+08:00
description = '之前搞了个 OrangePi R1 Plus LTS 作为软路由, 但是它也有一些标准的外部接口, 所以试着在 Linux 上操作它们吧'
draft = false
tags = ['OrangePi', 'Linux', '嵌入式', 'I2C']
+++

# 在香橙派 R1+ LTS 上使用 GPIO 与 I2C

## 背景

之前买了一块儿 OrangePi R1 Plus LTS 作为软路由, 最近无聊时翻阅它的手册, 发现它自带了一些 GPIO, 其中包括一个 TWI/I2C, 一个 UART, 于是尝试利用这些接口增加一些外设, 比如在显示屏上显示系统状态与 IP 地址, 以及控制风扇转速等. 由于之前没有在 Linux 下搞过这些东西 (只在真 · 单片机上做过), 于是正好学习在 Linux 下操作外设的方法.

## 在 Armbian 中启用 I2C

和官方手册略有不同的是, 在 Armbian 中启用 I2C 时对应的设备名不同. 不过我们也不需要手动添加, 在 `armbian-config` 的 `System > Hardware` 里面找到 `rk3328-i2c0`, 启用即可. 这实际上就是在 `/boot/armbianEnv.txt` 中增加一行

```plaintext
overlays=rk3328-i2c0
```

和手册上的名字不一样, 所以直接添加上面这行也行. 重启之后应当能看见 `/dev/i2c-0`

## 编译 WiringOP

WiringOP 就是板子开发商 Xunlong 给自家的 OrangePi 适配的 WiringPi. 这里没有什么不同, 直接按照手册下载编译即可:

```bash
git clone https://github.com/orangepi-xunlong/wiringOP
cd wiringOP
./build clean
./build 
```

## 确认 I2C 是否启用

将 I2C 设备接上, 板子的 13 Pin 接口从网口侧数, 定义分别如下:

> VCC  GND  SDA  SCL(SCK)

之后运行 `sudo i2cdetect -y 0`, 理论上应当能看到连接的设备地址. 例如对于地址为 `0x77` 的 BMP180 来说, 应当是这样的:

```bash
horizon@horizonpi-r1plus-lts:~$ sudo i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- 77
```

**如果你看到的是在 `0x18` 的位置上有 `UU`, 说明你选择了错误的 I2C 接口, 或者没有正确启用 I2C (对于这块板子, 你应当能在 `/dev/` 下看到两个 I2C 设备, 但只有 0 号是可使用的).**

或者, 此时运行 `gpio readall` (**先编译 WiringOP !**), 应该看到如下:

```bash
horizon@horizonpi-r1plus-lts:~$ gpio readall
 +------+-----+----------+--------+---+  R1 Plus +---+--------+----------+-----+------+
 | GPIO | wPi |   Name   |  Mode  | V | Physical | V |  Mode  | Name     | wPi | GPIO |
 +------+-----+----------+--------+---+----++----+---+--------+----------+-----+------+
 |      |     | 5V       |        |   |  1 || 2  |   |        | GND      |     |      |
 |   89 |   0 | SDA.0    |   ALT2 | 1 |  3 || 4  | 1 | ALT2   | SCK.0    | 1   | 88   |
 |  100 |   2 | TXD.1    |   ALT5 | 1 |  5 || 6  | 1 | ALT5   | RXD.1    | 3   | 102  |
 |      |     |          |        |   |  7 || 8  |   |        |          |     |      |
 |      |     |          |        |   |  9 || 10 | 1 | ALT3   | GPIO3_C0 | 4   | 112  |
 |  103 |   5 | CTS.1    |   ALT5 | 1 | 11 || 12 | 1 | ALT5   | RTS.1    | 6   | 101  |
 |   66 |   7 | GPIO2_A2 |     IN | 1 | 13 || 14 |   |        |          |     |      |
 +------+-----+----------+--------+---+----++----+---+--------+----------+-----+------+
 | GPIO | wPi |   Name   |  Mode  | V | Physical | V |  Mode  | Name     | wPi | GPIO |
 +------+-----+----------+--------+---+  R1 Plus +---+--------+----------+-----+------+
 ```

Pin 3 和 4 应当在 `ALT2` 模式下, 如果是 `IN` 说明没有启用 I2C, 它们只能当作普通的 GPIO 使用.

## 读取 I2C 设备 (BMP180)

使用 Python 读取 BMP180 气压传感器, 先安装 `smbus` 库, 之后代码如下

```py
#!/usr/bin/python3

import smbus
import time
import math

address = 0x77  #I2c device BMP180
mode = 1 # mode Standard

# BMP180 registers
CONTROL_REG = 0xF4
DATA_REG = 0xF6
# Calibration data registers
CAL_AC1_REG = 0xAA
CAL_AC2_REG = 0xAC
CAL_AC3_REG = 0xAE
CAL_AC4_REG = 0xB0
CAL_AC5_REG = 0xB2
CAL_AC6_REG = 0xB4
CAL_B1_REG = 0xB6
CAL_B2_REG = 0xB8
CAL_MB_REG = 0xBA
CAL_MC_REG = 0xBC
CAL_MD_REG = 0xBE


def read_signed_16_bit( register): #Lit valeur signée 16 bits
        msb = bus.read_byte_data(address, register)
        lsb = bus.read_byte_data(address, register + 1)
        if msb > 127:
            msb -= 256
        return (msb << 8) + lsb
def read_unsigned_16_bit( register):  #Lit valeur non signée 16 bits
        msb = bus.read_byte_data(address, register)
        lsb = bus.read_byte_data(address, register + 1)
        return (msb << 8) + lsb

# Temperature (see BMP180 datasheet)

def get_raw_temp():
        # Start the measurement
        bus.write_byte_data(address, CONTROL_REG, 0x2E)
        # Wait 5 ms
        time.sleep(0.005)
        # Read the raw data from the DATA_REG, 0xF6
        raw_data = read_unsigned_16_bit(DATA_REG)
        # Return the raw data
        return raw_data
def get_temp():
        UT = get_raw_temp()
        X1 = ((UT - calAC6) * calAC5) / math.pow(2, 15)
        X2 = calMC*math.pow(2, 11)/(X1+calMD)
        B5 = X1 + X2
        temperature = ((B5 + 8) / math.pow(2, 4)) / 10
        return temperature

# Pressure
def get_raw_pressure():
        bus.write_byte_data(address, CONTROL_REG, 0x34 + (mode << 6))
        # Sleep for 8 ms.
        time.sleep(0.008)
        # Read the raw data from the DATA_REG, 0xF6
        MSB = bus.read_byte_data(address, DATA_REG)
        LSB = bus.read_byte_data(address, DATA_REG + 1)
        XLSB = bus.read_byte_data(address, DATA_REG + 2)
        raw_data = ((MSB << 16) + (LSB << 8) + XLSB) >> (8 - mode)
        return raw_data
def get_pressure():
        UP = get_raw_pressure()
        UT = get_raw_temp()
        #calculation idem temperature
        X1 = ((UT - calAC6) * calAC5) / math.pow(2, 15)
        X2 = (calMC * math.pow(2, 11)) / (X1 + calMD)
        B5 = X1 + X2
        B6 = B5 - 4000
        X1 = (calB2 * (B6 * B6 / math.pow(2, 12))) / math.pow(2, 11)
        X2 = calAC2 * B6 / math.pow(2, 11)
        X3 = X1 + X2
        B3 = (((calAC1 * 4 + int(X3)) << mode) + 2) / 4
        X1 = calAC3 * B6 / math.pow(2, 13)
        X2 = (calB1 * (B6 * B6 / math.pow(2, 12))) / math.pow(2, 16)
        X3 = ((X1 + X2) + 2) / math.pow(2, 2)
        B4 = calAC4 * (X3 + 32768) / math.pow(2,15)
        B7 = (UP - B3) * (50000 >> mode)
        if B7 < 0x80000000:
            pressure = (B7 * 2) / B4
        else:
            pressure = (B7 / B4) * 2
        X1 = (pressure / math.pow(2, 8)) * (pressure / math.pow(2, 8))
        X1 = (X1 * 3038) / math.pow(2, 16)
        X2 = (-7357 * pressure) / math.pow(2, 16)
        pressure = pressure + (X1 + X2 + 3791) / math.pow(2, 4)
        pressure=pressure/100 #hPa
        return pressure


# I2C bus 0

bus = smbus.SMBus(0)

# Calibration data variables
calAC1 = read_signed_16_bit(CAL_AC1_REG)
calAC2 = read_signed_16_bit(CAL_AC2_REG)
calAC3 = read_signed_16_bit(CAL_AC3_REG)
calAC4 = read_unsigned_16_bit(CAL_AC4_REG)
calAC5 = read_unsigned_16_bit(CAL_AC5_REG)
calAC6 = read_unsigned_16_bit(CAL_AC6_REG)
calB1 = read_signed_16_bit(CAL_B1_REG)
calB2 = read_signed_16_bit(CAL_B2_REG)
calMB = read_signed_16_bit(CAL_MB_REG)
calMC = read_signed_16_bit(CAL_MC_REG)
calMD = read_signed_16_bit(CAL_MD_REG)

print("Calibration:",calAC1,calAC2,calAC3,calAC4,calAC5,calAC6,calB1,calB2,calMB,calMC,calMD)
while True:
    print("Temperature: {0:0.1f} C".format(get_temp()))
    print('Pressure = {0:0.1f} hPa'.format(get_pressure()))
    time.sleep(0.5)
```

代码来自 [这里](https://f1atb.fr/index.php/2020/10/04/bmp180-and-orange-pi-zero/). 虽然很长, 但大部分内容都是在处理 BMP180 给出的数据; 从 I2C 读数据只有很少一点. 如果没啥问题, 应该是这样的

```bash
horizon@horizonpi-r1plus-lts:~$ sudo python3 bmp.py
Calibration: 9096 -1237 -14304 34330 25663 16155 6515 51 -32768 -11786 2416
Temperature: 26.3 C
Pressure = 1008.0 hPa
Temperature: 26.2 C
Pressure = 1007.9 hPa
Temperature: 26.2 C
Pressure = 1007.9 hPa
.........
```

第一行是 BMP180 的校准数据, 后面是获取到的温度和气压. 通过 I2C 接口也可以操作其他设备, 等我买的显示屏到了再继续吧.

## 关于 Armbian 和 OrangePi 的不算彩蛋的彩蛋

通过 SSH 连接到板子时, motd 会包含板子型号的 [ASCII ART](https://en.wikipedia.org/wiki/ASCII_art), 也就是字符画. 由于空间有限, 一些词会缩写, 所以就会...这样...

```bash
❯ ssh horizon@192.168.1.9
  ___  ____  _   ____  _   ____  _             _   _____ ____
 / _ \|  _ \(_) |  _ \/ | |  _ \| |_   _ ___  | | |_   _/ ___|
| | | | |_) | | | |_) | | | |_) | | | | / __| | |   | | \___ \
| |_| |  __/| | |  _ <| | |  __/| | |_| \__ \ | |___| |  ___) |
 \___/|_|   |_| |_| \_\_| |_|   |_|\__,_|___/ |_____|_| |____/

Welcome to Armbian 23.11.1 Bookworm with Linux 6.1.63-current-rockchip64

System load:   2%               Up time:       26 min
Memory usage:  18% of 976M      IP:            [DELETED]
CPU temp:      50°C             Usage of /:    4% of 58G
RX today:      133.1 KiB

Last login: Wed Jan 10 23:08:51 2024 from [DELETED]
```

`OrangePi` 会被缩写成 `OPi`.... 负责这个输出的文件是 `/etc/update-motd.d/10-armbian-header`, 不过我不打算改~
