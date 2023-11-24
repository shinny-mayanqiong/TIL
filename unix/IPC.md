# IPC

有一系列文章, 在此记录学习到东西。


## [IPC 简介](https://goodyduru.github.io/os/2023/09/08/ipc-introduction.html)

Inter-Process Communication(IPC) 进程间通信，“操作系统为进程提供的管理共享数据的机制”。

shell 中的管道 ｜ 就是一种 IPC 机制。ls 和 grep 在通信。

```
ls -l | grep *.txt
```

进程独立，进程间通信的几种 IPC 机制：

* [Named Pipe 命名管道](#ipc---named-pipes)
* [Unix Domain Sockets 域套接字](#ipc---unix-domain-sockets)
* Message Queues  消息队列
* Shared Memory 共享内存
* Memory-Mapped Files 内存映射文件

这个博客没有讨论到的其他进程间通讯方式：

* files 文件
* TCP Sockets (网络)
* eventfd (另一种机制，但进程不能独立（它们必须在代码中共享一个父进程）)

## [IPC - Named Pipes](https://goodyduru.github.io/os/2023/09/26/ipc-named-pipes.html)

### Anonymous Pipes 匿名管道

由内核创建和维护的内存缓冲区。

匿名管道是单向的。

写入此缓冲区的数据不会出现在磁盘上。

数据一旦被读取就会从缓冲区中删除。

匿名管道只有创建进程和其子进程可以使用。所以 shell 中的匿名管道可以用于两个子进程之间通讯。

### Named Pipes 命名管道

命名管道就是给匿名管道起了个名字，并引用它。

`mkfifo example-pipes` 创建一个名为 example-pipes 的命名管道，是一个在磁盘上的文件。

可以 `open`， `read`， `write`，`unlink`，`close` 等等。而且可以有权限，可以限制谁有权访问它。

常规文件和 FIFO 文件之间的区别在于它永远不能包含数据。它仅仅是一个引用 reference。

命名管道可以是双向的。

要至少有一个并发读取器，才会将字节写入命名管道缓冲区。一定要先创建读的 fd，再创建写的 fd。


Python 示例：

```python
# server

import os

def run():
    pipe_path = '/tmp/ping'
    os.mkfifo(pipe_path)
    fd = os.open(pipe_path, os.O_RDONLY)
    data = os.read(fd, 4).decode()
    while data != 'end':
        print(f"Server: Received {data}")
        data = os.read(fd, 4).decode()
    os.close(fd)
    os.unlink(pipe_path)
```

```python
import os

ROUNDS = 100

def run():
    pipe_path = '/tmp/ping'
    fd = os.open(pipe_path, os.O_WRONLY)
    i = 0
    while i != ROUNDS:
        os.write(fd, b'ping')
        print("Client: Sent ping")
        i += 1
    os.write(fd, b'end')
    os.close(fd)    
```

## [IPC - Unix Domain Sockets](https://goodyduru.github.io/os/2023/10/03/ipc-unix-domain-sockets.html)

### 网络基础知识

* 物理层（光纤、4G、5G 等）涉及通过无线电波和光等物理介质发送数据。
* 链路层（以太网、WiFi 等）在网络之间传输数据。
* 网络层（BGP、ICMP 等）关注通过最有效的路径路由数据。
* 传输层（TCP、UDP 等）将数据传输到正确计算机上的正确应用程序进程。
* 应用层（SMTP、HTTP、万种其他协议）处理用户和数据之间的交互。

Unix 域套接字建立在 Unix 传输层接口之上。也称为 BSD 套接字。

### BSD Sockets

许多网络库都使用 BSD 套接字，也可以用于进程间通信（IPC）。

### Unix Domain Sockets (UDS)

基于 socket 的 IPC 机制，用于两个进程之间通信，使用分配给每个套接字的缓冲区来进行消息传输。这些缓冲区是操作系统在创建套接字时设置的。

套接字文件，使用 open() 函数访问该文件将返回错误。这样做的结果是几乎所有应用程序都无法打开该文件。

好处是可以利用文件系统权限设置限制文件安全。

UDS 是一种双向 IPC 机制，两端都可以写入、读取消息。

### BSD Sockets vs. UDS

使用 TCP/UDP 进行 IPC 的一个优点是您可以使用您最喜欢的网络库，而不用摆弄 Sockets API。

使用 UDS 时，计算机知道是本地通信，因此不会执行使用 TCP 完成的整个握手和确认。此外，您的数据不会经过整个 IP 堆栈机制。不必这样做对于 UDS 来说是一个巨大的性能提升。

### 示例

```python
# 客户端

import socket

ROUNDS = 100

def run():
    server_address = './udsocket'
    # 第一个参数，套接字系列 AF_UNIX 表示创建 UDS 域套接字，AF_INET 和 AF_INET6 用于互联网通信的 IPv4 和 IPv6 通信
    # 第二个参数，套接字类型 SOCK_STREAM 表示可靠的双向通信(也用于 TCP)，SOCK_DGRAM 用于 UDP, SOCK_RAW 用于原始套接字。
    sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    sock.connect(server_address)

    print(f"Connecting to {server_address}")
    i = 0
    msg = "ping".encode()
    while i < ROUNDS:
        sock.sendall(msg)
        print("Client: Sent ping")

        data = sock.recv(16)
        print(f"Client: Received {data.decode()}")
        i += 1
    sock.sendall("end".encode())
    sock.close()
```

```python
# 服务端

import os
import socket


def run():
    server_path = './udsocket'
    # 删除套接字文件以防止“地址已在使用”错误
    try:
        os.unlink(server_path)
    except OSError:
        pass

    server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    server.bind(server_path)  # 套接字绑定到套接字文件

    server.listen(1)  # 设置为监听模式, 最多允许一个连接, 服务器允许的连接数
    msg = "pong".encode()
    while True:
        connection, address = server.accept()  # 接受连接，返回一个新的套接字对象和一个地址
        print(f"Connection from {address}")
        while True:
            data = connection.recv(16).decode()
            if data != "ping":
                break
            print(f"Server: Received {data}")
            connection.sendall(msg)
            print(f"Server: Sent pong")
        if data == "end":
            print("Server: Connection shutting down")
            connection.close()
        else:
            print(f"Server Error: received invalid message {data}, shutting down connection")
            connection.close()
```





