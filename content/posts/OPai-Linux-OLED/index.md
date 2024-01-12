+++
title = '嵌入式 Linux 外设 [Part 2] -- 点亮屏幕'
date = 2024-01-11T15:55:06+08:00
description = '之前我们跑通了硬件的 I2C, 接下来我们试着用 I2C 去点亮屏幕吧'
draft = false
+++

# 在香橙派 R1+ LTS 上使用 I2C 点亮 OLED 屏幕和 1602 LCD

在 [上一篇文章](https://horizonchaser.github.io/neo/posts/opai-linux-gpio/) 中, 我们成功通过 I2C 接口读到了传感器的数据. 光有输入怎么行, 接下来哦我们继续利用 I2C 接口整点输出, 比如...点亮显示屏!

正好今天显示屏终于到了, 一块儿 0.91 英寸的 128x32 OLED, 一块儿 1602 LCD, 以及转到 I2C 接口的转接板. 接下来就开干!

## 点亮 OLED 屏幕

手头这块儿 OLED 屏幕驱动芯片是 `SSD1306`, 具体的原理暂时先略过, 可以直接使用 `luma.oled` 库操作 (记得先安装).

```python
from luma.core.interface.serial import i2c
from luma.core.render import canvas
from luma.oled.device import ssd1306
import time

# 创建 I2C 设备, SSD1306 默认地址是 0x3C
serial = i2c(port=0, address=0x3C)

# 创建屏幕的驱动实例
device = ssd1306(serial, width=128, height=32, rotate=2)

# 开始往屏幕上绘图
# draw 是 Pillow 的实例, 它里面还有非常多的绘图 API
with canvas(device) as draw:
  draw.rectangle(device.bounding_box, outline="white", fill="black")
  draw.text((10, 40), "Hello World", fill="white")

# luma 在退出时会自动调用一次清屏, 速度太快会导致我们什么也看不到
# 所以需要等待
time.sleep(1000)
```

接线还是和之前一样, 不再赘述. 对设备的操作需要 root 权限, 如果一切正常, 你应当能看到屏幕四周有白色矩形, 内部就是 "Hello World".

### 让屏幕动起来

为了测试一下显示屏在 I2C 总线上的响应速度, 画个大的进度条试试:

```python
from luma.core.interface.serial import i2c
from luma.core.render import canvas
from luma.oled.device import ssd1306
import time

# 创建 I2C 设备, SSD1306 默认地址是 0x3C
serial = i2c(port=0, address=0x3C)

# 创建屏幕的驱动实例
device = ssd1306(serial, width=128, height=32, rotate=2)

# 通过不断向右侧扩展的矩形来作为进度条
for i in range(0, 128):
  with canvas(device) as draw:
    draw.rectangle((0,0,i,31), fill="white", outline="white")
    print(f"curr col: {i}")
    time.sleep(0.001)

# 这里不再等待, 直接退出
```

同样以 root 权限执行, 通过 `time` 命令可以看到需要 7.8 秒才能执行完... 这速度还是太慢了. 如果需要高速刷新的话, 可以使用 SPI 总线操作.

## 点亮 1602 LCD 屏幕

1602 本身并不是 I2C 接口的, 不过通过转接模块, 我们可以将其转为 I2C 接口操作. 这不仅能极大的简化我们的操作, 还能省下很多 I/O 接口.

### 转接的原理

转接板的核心是一块儿 `PCF8574` 芯片, 它可以将 I2C 总线操作转为 8 位的 IO 输出 -- 而 1602 本身恰好就是 8 条数据线. 因此我们通过 I2C 总线向转接板写数据, 实质上也就是在直接操作 1602 的数据线. 1602 的指令格式不再赘述, 很容易就能查到.

除了 8 位的数据线, 转接板还提供了 LCD 背光的供电, 以及对比度调节. 这些都极大的简化了我们的接线 -- 1602 原本需要十几个接口, 但现在和其他 I2C 设备一样, 只需要正负供电和 SCL SDA 四条线即可.

### 点亮

1602 的 I2C 接口版本也有很多现成的实现, 例如 [`i2clcd`](https://pypi.org/project/i2clcd/). 其实只要阅读库的源代码, 就可以发现它实际上也是直接向对应的 I2C 地址写指令, [比如设置背光](https://github.com/WuSiYu/python-i2clcd/blob/master/i2clcd/__init__.py#L111)

```python
def set_backlight(self, on_off):
    """
    Set whether the LCD backlight is on or off
    """
    self._backlight = on_off
    i2c_data = (self._last_data & 0xF7) + self._backlight * 0x08
    self._i2c_write(i2c_data)
```

直接运行官方例程, 轻松点亮~

## 总结

没啥好总结的, 调通 I2C 接口之后点亮屏幕并没有什么难度 (但不妨碍我水一篇 :). 

不过不得不说的是, 这 OLED 屏幕也太小了, 本来是打算拿来显示软路由的 IP 和状态的, 这下害得靠 1602 了...
