---
title: 如何在 C# 中使用串口通信
slug: "how-to-use-serial-in-csharp"
date: 2026-11-07T10:09:18+08:00
categories: 技术
tags:
  - 通信, 
  - csharp
draft: false
description: "介绍串口通信的基本原理与数据帧结构，讲解如何在 C# 中使用 System.IO.Ports 配置串口参数、收发字符串与字节流数据，并附带串口调试工具的实现思路。"
---

## 串口通信介绍

串口通信（Serial Communication）是一种通过单条数据线**逐位传输数据**的方式，常用于单片机、传感器、计算机等设备之间的数据交换。

**核心机制**是将数据拆分成比特流，通过单条线路按顺序发送。以发送字符 `A` 为例（ASCII 码 `65`，二进制 `01000001`），串口会在数据前加**起始位 (0)**，数据位按约定顺序发送，最后加**停止位 (1)**，接收方检测到起始位后按波特率定时采样还原数据。

一个完整的串口数据帧包含：

```txt
[起始位] + [数据位] + [校验位(可选)] + [停止位]
```

## 如何在 C#中使用

要想在 C#中使用，必须先安装 `System.IO.Ports`，这是因为**串口通信库**没有内置在标准库内，需要我们手动添加。

```shell
dotnet add package System.IO.Ports
```

### 创建串口通信的关键参数

当我们引入串口通信库后，会根据事先约定好的参数，创建串口连接。

而这些是需要事先约定好的参数，如下：

| 参数                 | 说明                                                         | 常见值                                                    |
| -------------------- | ------------------------------------------------------------ | --------------------------------------------------------- |
| 波特率（`BaudRate`） | 表示数据传输速率，即每秒钟传输的二进制位数（bps）。它是衡量通信速度的关键参数，通信双方必须设置为完全相同的波特率，否则会导致数据无法正确解析或出现乱码。 | 9600, 19200, 38400, 57600, 115200, 230400, 460800, 921600 |
| 数据位（`DataBits`） | 表示每个数据帧中有效数据的位数，即一个信息包里包含的实际数据位数量。它决定了每次传输的数据量大小，影响数据传输的效率和兼容性。 | 5, 6, 7, 8（8 位最常用）                                   |
| 校验位（`Parity`）   | 用于检测单比特错误的校验机制。通过在数据位后添加一个校验位，确保数据传输的完整性。奇校验确保 "1" 的总数为奇数，偶校验确保 "1" 的总数为偶数，无校验则不进行错误检测。 | None（无）、Odd（奇校验）、Even（偶校验）                 |
| 停止位（`StopBits`） | 用于表示单个数据帧结束的标志位，固定为高电平。它帮助接收方识别数据帧的结束位置，确保数据同步和帧边界识别，防止数据粘连。 | 1, 1.5, 2（1 位最常用）                                    |

**补充说明：**

- 通信双方的所有参数必须完全一致才能实现正常通信。
- 最常用的配置组合为：9600/115200 波特率、8 数据位、无校验、1 停止位（8 N 1）。

### 创建串口连接

到创建串口连接这一步，还需要了解一个参数：

| 参数                   | 说明                                                         | 常见值                                                       |
| :--------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 串口名称（`PortName`） | 指操作系统分配给物理或虚拟串行端口的设备标识符。它是应用程序与特定串口硬件建立连接的关键参数，用于唯一识别和访问对应的串行通信端口。不同的操作系统使用不同的命名规则来标识串口设备。 | **Windows 系统**：COM 1, COM 2, COM 3, COM 4... <br>**Linux 系统**：/dev/ttyS 0, /dev/ttyS 1, /dev/ttyUSB 0, /dev/ttyACM 0...<br> **macOS 系统**：/dev/cu.usbserial-XXXX, /dev/tty.usbmodemXXXX... |

简而言之，我们需要一个可正常使用的串口，来作为数据传输的出入口。

> [!NOTE]
> 在我做练习的过程当中，发现消费级笔记本电脑通常不再配备物理串口，但在工业控制领域工控机通常仍保留串口接口。如果没有物理串口，可以使用 USB 转串口适配器，或者在开发测试阶段使用虚拟串口软件创建配对的虚拟串口。
>
> 参考连接：[为什么通信要使用虚拟串口和串口助手？ - 知乎](https://zhuanlan.zhihu.com/p/7666669617)

#### 步骤 1：引入串口类的命名空间

```csharp
using System.IO.Ports;
```

#### 步骤 2：初始化串口

现在假设，接下来要使用的串口名称为 `COM1`，其余参数为：

- 波特率：9600
- 数据位：8
- 校验：无
- 停止位：1

```csharp
var sp = new SerialPort(
    portName: "COM1",
    baudRate: 9600,
    parity: Parity.None,
    dataBits: 8,
    stopBits: StopBits.One)
{
    // 设置编码方式为UTF-8
    Encoding = Encoding.UTF8,
    // 设置读写超时时间，统一为1秒
    ReadTimeout = 1000,
    WriteTimeout = 1000
};
```

同时设置数据编码方式和读写超时时间。

#### 步骤 3：打开串口

```csharp
sp.Open();
```

在这里可以添加 `try-catch`，来捕获异常，比如可能会发生的串口被占用等等。

```csharp
try
{
    sp.Open();
}
catch (Exception e)
{
    // 捕获并显示所有可能的串口异常
    // 常见异常：UnauthorizedAccessException（串口被占用）、
    //          IOException（串口不存在）、ArgumentOutOfRangeException（参数无效）
    Console.Out.WriteLine($"异常：{e.Message}");
}
```

### 读写操作

在 C#中串口的读与写，都可以使用两种类型的数据，分别是字符串和字节流。

#### 发送数据

直接写入字符串数据。使用 [`Write(string text)`](https://learn.microsoft.com/en-us/dotnet/api/system.io.ports.serialport.write?system-io-ports-serialport-write(system-string)) 将指定字符串数据发送。

```csharp
sp.Write("Hi");
```

直接写入字节流数据。使用 [`Write(byte[] buffer, int offset, int count)`](https://learn.microsoft.com/en-us/dotnet/api/system.io.ports.serialport.write) 写入指定数量的字节。

```csharp
byte[] data = Encoding.UTF8.GetBytes("Hi");
sp.Write(data, 0, data.Length);
```

- `buffer`：将要发送的字节数组
- `offset`：字节写入的偏移量，直接从字节数据中第几位开始向缓冲区复制。
- `count`：根据其值，确定每次写入多少位字节。

> 发送字节流数据时，使用了 `Encoding.UTF8` 工具类，可以将字符串或者字节数组，按照 UTF-8 编码转换为指定的类型。

#### 读取数据

直接获取字符串数据。使用 [`ReadExisting()`](https://learn.microsoft.com/en-us/dotnet/api/system.io.ports.serialport.readexisting?system-io-ports-serialport-readexisting) 将会读取 接收缓冲区中当前可用的所有数据 ，按照预先设置的编码将字节流解码为 string 类型返回。

```csharp
string data = sp.ReadExisting();
```

直接获取字节流数据。使用 [`Read(byte[] buffer, int offset, int count)`](https://learn.microsoft.com/en-us/dotnet/api/system.io.ports.serialport.read) 读取缓冲区内的数据。

```csharp
var bytesToRead = sp.BytesToRead;
var buffer = new byte[bytesToRead];
sp.Read(buffer, 0, bytesToRead);
```

- `BytesToRead`：返回的是调用时缓冲区中的字节数。

在这里，我们直接将缓冲区内的数据全部读取，在数据量不大的时候这么干。

#### 数据接收事件

串口类提供了 [DataReceived](https://learn.microsoft.com/en-us/dotnet/api/system.io.ports.serialport.datareceived?system-io-ports-serialport-datareceived) 事件，来实时接收串口数据。每当有数据抵达时，事件就会触发。

```csharp
// 注册 DataReceived 事件
sp.DataReceived += (sender, args) =>
{
	var s = (SerialPort)sender;
    string received = s.ReadExisting(); // 读取所有可用数据
    Console.WriteLine($"接收：{received}");
};
```

### 关闭串口连接

关闭串口连接。

```csharp
sp.Close();
```

销毁串口对象。

```csharp
sp.Dispose();
```

## 实践

为了进一步掌握串口通信，我编写了一个功能简单的串口调试工具，具有收发数据的功能，可收发字符串或 16 进制的数据。

接下来，主要展示收发数据的核心逻辑，其余功能的细节，可查阅[此仓库](https://github.com/qisork/SerialDebugTool)。

### 发送数据

关于发送哪种数据，由 `ToggleButton` 按钮控制。通过命令绑定的方式，将 `ToggleButton` 的 `IsChecked` 作为参数传递，如果为真，则采用 16 进制发送数据，否则反之。

```csharp
[ObservableProperty]
private string _sendData = string.Empty;

[RelayCommand]
private void Send(bool isHexMode)
{
    // 验证串口是否已打开
    if (_serialPort is not { IsOpen: true })
    {
        MessageBox.Show("串口未打开");
        return;
    }

    // 根据模式选择发送方式
    try
    {
        if (!isHexMode)
        {
            // 发送文本数据
    		_serialPort?.Write(SendData);
        }
        else
        {
            // 分割十六进制字符串，支持空格、回车、换行作为分隔符
            var hexValues = SendData.Split([' ', '\r', '\n'], StringSplitOptions.RemoveEmptyEntries);

            // 将分割好的数组转换为 byte 数组
            var bytes = hexValues
                .Select(hex => Convert.ToByte(hex, 16))
                .ToArray();
			
            // 发送字节数据
            _serialPort?.Write(bytes, 0, bytes.Length);
        }
    }
    catch (Exception e)
    {
        MessageBox.Show($"发送数据失败: {e.Message}");
    }
}
```

> 在这里使用了 CommunityToolkit.Mvvm 框架，来简化 MVVM 模式的开发。

### 读取数据

在这里，也是通过 `ToggleButton` 按钮，来控制数据类型。不过由于是使用“数据接收事件”，来实时接收数据，就不能直接将 `IsChecked` 的值作为命令参数传递，而是使用数据绑定，将其值同步到 ViewModel 中指定的属性上。

接下来，在构造函数中，添加“数据接收事件”。

```csharp
_serialPort.DataReceived += OnSerialDataReceived;
```

数据接收事件：

```csharp
/// <param name="sender">触发事件的串口对象</param>
/// <param name="args">串口数据接收事件参数</param>
private void OnSerialDataReceived(object sender, SerialDataReceivedEventArgs args)
{
    // 类型转换
    var sp = (SerialPort)sender;

    // 读取缓冲区中的所有数据
    var bytesToRead = sp.BytesToRead;
    var buffer = new byte[bytesToRead];
    sp.Read(buffer, 0, bytesToRead);
    
    string data;
    
    if (IsReceiveHexMode)
    {
        // 十六进制模式：将字节数组转换为空格分隔的大写十六进制字符串
        data = string.Join(" ", buffer.Select(b => b.ToString("X2")));
    }
    else
    {
        // 文本模式：使用UTF-8解码字节数组
        data = Encoding.UTF8.GetString(buffer);
    }
}
```

### 结束语

在这里提供的代码实现，都是简略的，是为了直观的呈现最核心的部分。

## 参考连接

- [C# SerialPort 使用详解 | CSDN博客](https://blog.csdn.net/qq_39847278/article/details/146395034)
- [超简单的一种通信，2分钟搞懂，串口通讯的工作原理 | 哔哩哔哩](https://www.bilibili.com/video/BV1y34y147s5)
- [串口通信（UART）完全指南 |  掘金](https://juejin.cn/post/7563549246658953251)
