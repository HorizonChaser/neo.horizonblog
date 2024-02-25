+++
title = 'Mesa Driver Breaks VMWare'
date = 2024-02-25T11:24:25+08:00
description = '新版本的 mesa 驱动和 VMWare 有冲突, 导致 3D 加速卡死 -- 降级可解'
draft = false
tags = ['Linux', '虚拟机', 'debug']
+++

# 新版本的 mesa 驱动与 VMWare 3D 加速的冲突

## 起因

昨天打开虚拟机中的 Fedora, 发现 进到 gdm 后直接黑屏, 鼠标移动到一些控件上的时候也许能显示出来. 同时 VMX 的 CPU 占用率一直在 ~20% 左右. 约几十秒后整个虚拟机卡死, 关闭时 VMWare 也会停止响应, 只能通过终止 VMX 退出. 查看事件查看器发现有大量的 LiveKernelEvent 141, `mksandbox` 崩溃.

一开始以为显卡或者显存出问题了, 跑了几轮 3DMark 下来也没发现啥; 把显卡驱动降级回 546.33 也没有什么改善. 显卡这边先排除了.

## 经过

折腾了半天打算重装系统的时候, 发现 VMWare 论坛上有一些人最近 (23年底) 也遇到了[同样的问题](https://communities.vmware.com/t5/VMware-Workstation-Player/Linux-freezes-VMware-so-much-I-have-to-end-it-with-task-manager/td-p/3007215), 表示是 6.5 及以上的新版内核或者新的 mesa 驱动导致的.

为了验证这点, 新开了一个 Debian 12.5 的虚拟机, 相关组件的版本都比较旧, 一切丝滑, 没任何问题...

## 解决

对于 Fedora 虚拟机:

1. 关闭 3D 加速, 进入系统, 此时会采用软件渲染
2. 安装 `versionlock` 插件 (若无): `sudo dnf install 'dnf-command(versionlock)'`
3. 降级 mesa 驱动: `sudo dnf downgrade 'mesa*'`
4. 固定 mesa 版本: `sudo dnf versionlock add 'mesa*'`, 等新版本修好了再更新吧

关闭虚拟机, 打开 3D 加速, 然后继续丝滑吧 (
