---
title: 如何在Linux中，为Flatpak版本的Edge浏览器导入证书
date: 2025-10-30T16:54:42+08:00
tags:
  - linux
  - flatpak
  - edge
  - certificate
  - 指南
draft: false
description: "本文介绍了在 Linux 系统中为 Flatpak 版本的 Edge 浏览器导入证书的两种方法，解决了 Flatpak Edge 浏览器因缺少证书管理选项，而导致的问题。"
summary: "当使用 Flatpak 版本的 Edge 浏览器时，可能会遇到缺少证书管理功能的问题。本文提供了两种解决方案：1）通过 edge://certificate-manager/localcerts 内部链接图形化导入证书；2）将证书文件复制到系统信任目录 /etc/pki/ca-trust/source/anchors/ 并执行 update-ca-trust 命令。第二种方法特别适用于第一种方法出现导入错误的情况。"
---

当我在使用 [Watt Toolkit](https://steampp.net/) 时，发现 Flatpak 版本的 Edge 浏览器并没有“证书管理”的选项，只能通过 Edge 的内部链接（`edge://certificate-manager/localcerts`）访问“证书管理器”。

经过研究，要为 Edge 导入证书，有两种方法：

1. 通过 [Edge 内部链接](edge://certificate-manager/localcerts)，使用图形化的方式导入，与正常的方法无异。
2. 将证书复制到 `/etc/pki/ca-trust/source/anchors/` 目录下。

> `/etc/pki/ca-trust/source/anchors/` 目录是 Fedora/Red Hat 类 Linux 中用于管理系统级受信任根证书的核心目录。
>
> 而在这个路径因 Linux 发行版不同而不同，如果使用的是其他 Linux 发行版，需要找到对应的目录。

## 使用方法 2 导入证书的步骤：

{{% steps %}}

### 复制证书文件

将 PEM 或 DER 格式的证书文件（如 `my-ca.cer`）复制到该目录：

```bash
sudo cp /path/to/my-ca.cer /etc/pki/ca-trust/source/anchors/
```

> 在这里请将证书替换为自己的证书所在位置

### 更新信任库

执行 `update-ca-trust` 命令，使系统重新生成整合后的信任库文件，供各应用程序使用：

```bash
sudo update-ca-trust
```

{{% /steps %}}

## 补充信息

在使用方法 1 导入时，可能出现导入错误的情况，这时可使用方法 2 进行导入。