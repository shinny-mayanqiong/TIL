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
* [Unix Signals 信号](#ipc---unix-signals)
* [Message Queues 消息队列](#ipc---message-queues)
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

## [IPC - Unix Signals](https://goodyduru.github.io/os/2023/10/05/ipc-unix-signals.html)

其他 IPC 机制是进程间传递数据(bytes)，进程需要解析数据后才能知道要做什么。而 Signals 是由操作系统决定应该做什么。


### Processes and Process Group 进程和进程组

`ps -o "pid,tty,time,command,pgid"` 可以列出所有的进程以及进程组。

默认情况下，每个进程都属于一个进程组，进程组 ID 与进程 ID 相同。shell 运行脚本的时候，会把创建的进程分配给子进程。

### 系统调用 kill

`kill -special_number pid` 命令可以向进程发送信号。默认情况下，`kill` 发送 `SIGTERM` (-8) 信号，这是一个终止信号，进程可以忽略它。

`kill 0` pid 设置为 0 以杀死进程组中的所有进程。

`kill -l` 可以列出所有信号。

### Signals 信号

信号是操作系统发送到进程的标准化消息。[wikipedia](https://en.wikipedia.org/wiki/Signal_(IPC)#POSIX_signals)

信号具有高优先级，进程必须中断正常流程，优先处理这些信号。

将信号发送到进程的两种方式。
+ raise 函数只是一个向自身发送信号的进程。
+ kill 命令，我很快就会讨论它的等效系统调用函数。操作系统内核还可以通过直接操作进程结构来发送信号

每个信号在进程中都必须有一个处理函数。每当进程收到信号时就会执行此函数。该函数可以在内核或用户级代码中定义。当操作系统启动一个新的应用程序进程时，它会为其每个信号对象分配默认处理程序。某些信号的默认处理程序会终止进程。其他一些默认处理程序不执行任何操作，即信号被忽略。

信号的默认处理程序可以用常量 SIG_DFL 引用。您可以在此处查看[默认信号操作的列表](https://github.com/torvalds/linux/blob/7de25c855b63453826ef678420831f98331d85fd/kernel/signal.c#L1722)。

信号默认处理程序可以更改为不同的处理程序。该处理程序可以是定义的函数或 SIG_IGN(忽略该信号) 。 

如果希望处理一次信号并立即重置默认处理程序。可以通过在定义的处理程序中将信号的处理程序函数设置为 SIG_DFL 来做到这一点。

`SIGKILL` 和 `SIGSTOP` 信号不能被停止或忽略，除此之外，几乎所有信号的处理程序都可以更改。

如果需要两个进程在不知道彼此 pid 的情况下使用 Signals 进行通信，可以用进程组来创建两个进程，然后用 `kill -special_number 0` 来互相发送信号。

### 示例代码

```shell
sh run.sh
```

```shell
# run.sh
trap "" USR1 USR2 QUIT
python3 server.py & python3 client.py

# trap 用于在 bash 脚本中捕获信号并执行代码
# trap [command] [signal1 signal2 ...]
# "" 表示忽略信号
# 因为 bash 脚本也是一个进程，所以可以用 trap 来捕获信号，并忽略信号
```

```python
# client
import os
import signal

i = 0
ROUNDS = 100


def print_pong(signum, frame):
    global i
    os.write(1, b"Client: pong\n")
    i += 1


def run():
    signal.signal(signal.SIGUSR1, print_pong)
    while i < ROUNDS:
        os.kill(0, signal.SIGUSR2)
    os.kill(0, signal.SIGQUIT)


run()
```

```python
# server
import os
import signal

should_end = False


def end(signum, frame):
    global should_end
    os.write(1, b"End\n")
    should_end = True


def print_ping(signum, frame):
    os.write(1, b"Server: ping\n")
    os.kill(0, signal.SIGUSR1)


def run():
    signal.signal(signal.SIGUSR2, handler=print_ping)
    signal.signal(signal.SIGQUIT, end)

    while not should_end:
        pass


run()
```

## [IPC - Message Queues](https://goodyduru.github.io/os/2023/11/13/ipc-message-queues.html)

分为两种类型 - System V 和 POSIX。

[Message Queues](https://github.com/torvalds/linux/blob/6bc986ab839c844e78a2333a02e55f02c9e57935/ipc/msg.c#L49) 是消息的链接列表。

操作系统可以维护多个已发送消息的列表，每个列表都由唯一的整数标识符引用。消息通过附加到列表来发送，并通过从列表头部弹出来接收。

消息队列由操作系统内核管理并存储在内存中。允许异步通信。

运行 `ipcs -q` 查看操作系统中的所有消息队列。


### System V Message Queues

**生成密钥**

在创建或访问消息队列之前，需要确定性地生成唯一密钥。所有应用程序进程必须使用相同的密钥才能通过同一队列进行通信。

生成密钥的推荐方法是调用 `ftok` 函数。该函数接受文件路径和整数。文件路径必须是现有文件，否则将返回错误。推荐的文件路径可以是应用程序配置文件。只要文件未被删除并重新创建， `ftok` 函数将始终返回相同的结果。碰撞可能会发生，但发生的可能性很小。

**创建或访问消息队列**

访问或创建消息队列是使用 `msgget` 函数完成的。

该函数接受一个键和一个标志参数。创建消息队列是通过在标志中指定 `IPC_CREAT` 来完成的。消息队列权限在 flag 参数中定义。该权限与文件权限的格式相同。

例如，创建一个消息队列，只授予有效用户写权限，其他用户读权限，则通过执行 `msgget(key, IPC_CREAT | 0644)` 来完成。这将创建一个消息队列，其中用户 ID 未设置为队列所有者 ID 的进程可以接收消息但无法发送消息。

`msgget` 函数返回消息队列标识符，在从队列发送和接收消息时使用该标识符。

**接受和发送消息**

使用 `msgsnd` 和 `msgrcv` 函数以**字节**形式发送和接收的。 

消息队列由大小限制，可以配置。如果队列已满， `msgsnd` 将阻塞。可以通过在标志参数中指定 `IPC_NOWAIT` 来防止阻塞，直接返回错误。

`msgrcv` 函数接收消息，参数【类型】，确定进程是否要读取任何消息 (0)、特定消息类型（正整数）或特定消息组（负整数）。此函数从队列中删除消息并将其复制到提供的消息缓冲区参数。

如果队列为空/没有指定类型的消息， `msgrcv` 函数将默认阻塞。在标志参数中指定 `IPC_NOWAIT` 可以防止阻塞，直接返回错误。在 Linux 中，如果没有该类型的消息，并且在标志参数中指定了 `MSG_EXCEPT` ，则可以读取队列中的第一条消息。

消息队列可以通过 `msgctl` 函数删除和配置。该函数还允许读取消息队列元数据。

### 示例代码

Python 不提供开箱即用的消息队列支持。使用了 [sysv-ipc](https://pypi.org/project/sysv-ipc/)

```python
# server

import os
import sysv_ipc


def run():
    path = '/tmp/example'
    fd = os.open(path, flags=os.O_CREAT)  # create file
    os.close(fd)
    # key = sysv_ipc.ftok(path, 42)
    # print("key:", key)
    key = 123456
    mq = sysv_ipc.MessageQueue(key, flags=sysv_ipc.IPC_CREAT, mode=0o644)
    msg_type = 10
    data, _ = mq.receive(type=msg_type)
    data = data.decode()
    while data != 'end':
        print(f"Server: Received {data}")
        mq.send(b"pong", type=(msg_type + 1))
        print(f"Server: Sent pong")
        data, _ = mq.receive(type=msg_type)
        data = data.decode()
    os.unlink(path)
    mq.remove()


if __name__ == '__main__':
    run()
```

```python
# client
import os
import sysv_ipc

ROUNDS = 100


def run():
    # path = '/tmp/example'
    # fd = os.open(path, flags=os.O_CREAT)
    # os.close(fd)
    # key = sysv_ipc.ftok(path, 42)
    key = 123456
    mq = sysv_ipc.MessageQueue(key, flags=sysv_ipc.IPC_CREAT, mode=0o644)
    msg_type = 10
    i = 0
    while i != ROUNDS:
        mq.send(b"ping", type=msg_type)
        print("Client: Sent ping")
        data, _ = mq.receive(type=(msg_type+1))
        data = data.decode()
        print(f"Client: Received {data}")
        i += 1
    mq.send(b"end", type=msg_type)


if __name__ == '__main__':
    run()
```











