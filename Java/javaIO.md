# Socket
## 什么是socket
在网络编程中，网络上的两个程序通过一个双向的通信连接实现数据的交换，这个连接的一端称为一个socket。

socket套接字是通信的基石，是支持TCP/IP协议的网络通信的基本操作单元。它是网络通信过程中端点的抽象表示，包含进行网络通信必须的五种信息：连接使用的协议，本地主机的IP地址，本地进程的协议端口，远程主机的IP地址，远程进程的协议端口。

## Socket原理
Socket实质上提供了进程通信的端点。进程通信之前，双方首先必须各自创建一个端点，否则是没有办法建立联系并相互通信的。正如打电话之前，双方必须各自拥有一台电话机一样。
套接字之间的连接过程可以分为三个步骤：服务器监听，客户端请求，连接确认。

1. 服务器监听：是服务器端套接字并不定位具体的客户端套接字，而是处于等待连接的状态，实时监控网络状态。
2. 客户端请求：是指由客户端的套接字提出连接请求，要连接的目标是服务器端的套接字。为此，客户端的套接字必须首先描述它要连接的服务器的套接字，指出服务器端套接字的地址和端口号，然后就向服务器端套接字提出连接请求。
3. 连接确认：是指当服务器端套接字监听到或者说接收到客户端套接字的连接请求，它就响应客户端套接字的请求，建立一个新的线程，把服务器端套接字的描述发给客户端，一旦客户端确认了此描述，连接就建立好了。而服务器端套接字继续处于监听状态，继续接收其他客户端套接字的连接请求。

![img](../img/socket.png)

# TCP实现
```
public static void main(String[] args) {
    try {
        ServerSocket serverSocket = new ServerSocket();
        Socket socket = serverSocket.accept();
        new TcpServerThread(socket).run();
    } catch (IOException e) {
        e.printStackTrace();
    }
}


class TcpServerThread extends Thread {
    private Socket socket;

    public TcpServerThread(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        try {
            InputStream inputStream = socket.getInputStream();
            OutputStream outputStream = socket.getOutputStream();

            int len = 0;
            byte[] bufferArr = new byte[1024];
            len = inputStream.read(bufferArr);
            String content = new String(bufferArr, 0, len);
            String sendData = new String(String.valueOf(content.length()));
            outputStream.write(sendData.getBytes(), 0, sendData.length());
            outputStream.flush();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}


class TcpClient {
    public void connect() {
        try {
            Socket socket = new Socket("127.0.0.1", 650000);
            InputStream inputStream = socket.getInputStream();
            OutputStream outputStream = socket.getOutputStream();
            byte[] sendData = "hi".getBytes();
            outputStream.write(sendData, 0, sendData.length);
            outputStream.flush();

            int len = 0;
            byte[] bufferArr = new byte[1024];
            len = inputStream.read(bufferArr);
            String content = new String(bufferArr, 0, len);
            System.out.println(content);
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

# UDP实现
```
class UDPServer {
    public void udpConnect() throws IOException {
        DatagramSocket socket = new DatagramSocket(65001);
        byte[] buff = new byte[100];
        DatagramPacket packet = new DatagramPacket(buff, buff.length);
        // 接受客户端发送过来的内容，并将内容封装成DatagramPacket对象
        socket.receive(packet);
        byte[] data = packet.getData();
        String content = new String(data, 0, packet.getLength());
        System.out.println(content);
        byte[] sendedContent = String.valueOf(content.length()).getBytes();
        DatagramPacket packetToClient = new DatagramPacket(sendedContent, sendedContent.length, packet.getAddress(), packet.getPort());
        socket.send(packetToClient);
    }
}

class UDPClient {
    public void udpConnect() throws IOException {
        // 发送数据报
        DatagramSocket socket = new DatagramSocket();
        byte[] buf = "hi".getBytes();
        // 将IP地址封装成InetAddress对象
        InetAddress address = InetAddress.getByName("127.0.0.1");
        // 将要发送给服务端的数据封装成DatagramPacket
        DatagramPacket packet = new DatagramPacket(buf, buf.length, address, 65001);
        socket.send(packet);

        // 接收服务端数据报
        byte[] data = new byte[100];
        DatagramPacket receivePacket = new DatagramPacket(data, data.length);
        socket.receive(receivePacket);
        String content = new String(receivePacket.getData(), 0, receivePacket.getLength());
        System.out.println(content);

    }
}
```









