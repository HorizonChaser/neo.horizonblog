+++
title = 'HoriOS Part01 -- 实现一个简单的 UEFI Shell 和 Bootloader'
date = 2024-01-21T22:30:39+08:00
description = '尝试写一个基于 UEFI 的 x64 下的操作系统, 先从 UEFI 本身开始, 写一个简单的 Shell 和引导程序来了解 UEFI API 吧!'
draft = false
+++

# HoriOS Part01: UEFI Shell & bootloader

这篇文章已经咕咕咕好几天了, ~~绝对不是因为幻兽帕鲁太好玩了什么的 (捂脸~~

不过至少现在是开始写了, 大概

## 关于 HoriOS Project

HoriOS 是一个 (还没开始写的) 基于 x64 (或者 RISC-V) 与 UEFI 的, 支持多任务与虚拟内存的, 带有一个 Shell (以及相应的工具) 的玩具操作系统. 也许会支持文件系统和网络这些东西.

## UEFI 环境

UEFI 和传统的 BIOS (也许这里应该称作CSM) 的功能类似, 不过比起后者的历史包袱要小得多 -- 可以直接以 C 语言调用其提供的 API, 比写汇编什么的好多了. UEFI 本身的标准是开放的, 在 [uefi.org](uefi.org) 上可以直接找到.

UEFI 提供的 "功能" 被称作协议 (protocol), 一部分协议会在系统表 (`EFI_SYSTEM_TABLE`, 这是一个大的结构体, 也许可以理解为平台本身, 因为几乎所有的协议都需要系统表作为参数) 中提供. 而其余的协议, 则需要通过协议本身的 UUID 以及一个函数获取入口.

UEFI 的可执行程序本身就是 [PE32+](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format) 的格式 (大概是因为 UEFI 的前身 EFI 是来自安腾那边, 微软和英特尔一起搞的吧), 对于 GCC, `--subsystem,10` 即可指定该格式. UEFI 程序的入口固定是 `efi_main`, 需要手动指定; 由于我们不需要标准的库和头文件, 所以还需要 `-nostdinc -nostdlib`. 其余的参数就和普通的程序没有区别了.

## 实现一个 UEFI 下的 Shell

> To be noticed, 这里讨论的主题不是 UEFI Shell (UEFI 自带的 Shell), 而是在 UEFI 环境下实现一个它的类似物.

### 文本的输入输出

为了实现 Shell, 我们需要实现两样东西: 文本输入和输出. 幸运的是, 这两个协议都在系统表中有定义, 这样调用起来就方便很多. [系统表的定义](https://blog.gztime.cc/posts/2022/2430028/) 如下:

```C
typedef struct {
  EFI_TABLE_HEADER                 Hdr;
  CHAR16                           *FirmwareVendor;
  UINT32                           FirmwareRevision;
  EFI_HANDLE                       ConsoleInHandle;
  EFI_SIMPLE_TEXT_INPUT_PROTOCOL   *ConIn;
  EFI_HANDLE                       ConsoleOutHandle;
  EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL  *ConOut;
  EFI_HANDLE                       StandardErrorHandle;
  EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL  *StdErr;
  EFI_RUNTIME_SERVICES             *RuntimeServices;
  EFI_BOOT_SERVICES                *BootServices;
  UINTN                            NumberOfTableEntries;
  EFI_CONFIGURATION_TABLE          *ConfigurationTable;
} EFI_SYSTEM_TABLE;
```

可以看到, 我们需要的文本输入和输出都在系统表中提供了. `EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL` 本身也是一个结构体, [定义](https://uefi.org/specs/UEFI/2.10/12_Protocols_Console_Support.html#efi-simple-text-output-protocol)) 如下:

```c
typedef struct _EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL {
 EFI_TEXT_RESET                           Reset;
 EFI_TEXT_STRING                          OutputString;
 EFI_TEXT_TEST_STRING                     TestString;
 EFI_TEXT_QUERY_MODE                      QueryMode;
 EFI_TEXT_SET_MODE                        SetMode;
 EFI_TEXT_SET_ATTRIBUTE                   SetAttribute;
 EFI_TEXT_CLEAR_SCREEN                    ClearScreen;
 EFI_TEXT_SET_CURSOR_POSITION             SetCursorPosition;
 EFI_TEXT_ENABLE_CURSOR                   EnableCursor;
 SIMPLE_TEXT_OUTPUT_MODE                  *Mode;
} EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL;
```

东西真多, 但我们暂时只需要关注 `OutputString`, 它的定义如下 (我保证这是在输出部分的最后一次):

```c
typedef
EFI_STATUS
(EFIAPI *EFI_TEXT_STRING) (
 IN EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL    *This,
 IN CHAR16                             *String
 );
```

还是很直观的, `This` 是指向输出协议自己的指针, `String` 就是要输出的字符串, 以 `\0` 结尾. `CHAR16` 说明这是个 UCS-2 编码的 Unicode 字符串 (不过特定的 UEFI 实现本身可能并不支持输出 Unicode 中的全部字符... QEMU 的 TianoCore 就不支持中文).

和在其它语言中一样, 输入总是比输出复杂一些. `EFI_SIMPLE_TEXT_INPUT_PROTOCOL` 定义如下:

```c
typedef struct _EFI_SIMPLE_TEXT_INPUT_PROTOCOL {
 EFI_INPUT_RESET                       Reset;
 EFI_INPUT_READ_KEY                    ReadKeyStroke;
 EFI_EVENT                             WaitForKey;
} EFI_SIMPLE_TEXT_INPUT_PROTOCOL;
```

`ReadKeyStroke` 函数可以让我们获得在按下一个按键时的输入, 不过它是非阻塞的, 在没有按键按下的时候会返回 `EFI_NOT_READY`, 反之返回的是 `EFI_SUCCESS` (值为 0). 它通过一个 `EFI_INPUT_KEY` 类型的指针参数传回按键值, 定义如下:

```c
struct EFI_INPUT_KEY {
    unsigned short ScanCode;
    unsigned short UnicodeChar;
};
```

可以看到输入值分为了两部分, `ScanCode` 对应那些不在 Unicode 范围内的按键, 比如 Esc 键什么的, 当输入值为 Unicode 范围内时其值为零; 反之对 `UnicodeChar` 同理.

有一个细节需要注意, UEFI 中的输入默认不会回显, 也就是说如果程序不主动输出, 用户看不到自己输入. 因此如果在我们的 Shell 中不主动进行回显, 用户可能会非常困惑.

### Shell: REPL

> REPL: Read, Evaluate, Print Loop, 即"读取, 求值, 打印"的循环

简单起见, 我们的 Shell 本质上就是一个大的 (无限) 循环.

1. 读取用户的按键, 回显到屏幕上, 直到用户按下回车键, 完成一条输入
2. 解析输入内容, 然后执行对应的操作
3. 打印操作的结果 (如果有的话)
4. 回到读取的状态 1

