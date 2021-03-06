# Java基于TCP协议的编程

TCP/IP通信协议是一种可靠的网络协议，它在通信的两端各建立一个Socket，从而在通信的两端之间形成网络虚拟链路。一旦建立了虚拟的网络链路，两端的程序就可以通过虚拟链路进行通信。Java对基于TCP协议的网络通信提供了良好的封装，Java使用Socket对象来代表两端的通信端口，并通过Socket产生IO流来进行网络通信。

## 一、TCP协议基础

IP协议是Internet上使用的一个关键协议，它的全称是Internet Protocol，即Internet协议，通常简称IP协议。通过使用IP协议，从而使Internet成为一个允许连接不同类型的计算机和不同操作系统的网络。要使两台计算机彼此能进行通信，必须使两台计算机使用同一种“语言”，IP协议只保证计算机能发送和接收分组数据。IP协议负责将消息从一个主机传送到另一个主机，消息在传送的过程中被分割成一个个的小包。尽管计算机通过安装IP软件，保证了计算机之间可以发送和接收数据，但IP协议还不能解决数据分组在传输过程中可能出现的问题。因此，若要解决可能出现的问题，连上Internet的计算机还需要安装TCP协议来提供可靠并且无差错的通信服务。

TCP协议被称作一种端对端协议。这是因为它对两台计算机之间的连接起了重要作用——当一台计算机需要与另一台远程计算机连接时，TCP协议会让它们建立一个连接：用于发送和接收数据的虚拟链路。TCP协议负责收集这些信息包，并将其按适当的次序放好传送，接收端收到后再将其正确地还原。TCP协议保证了数据包在传送中准确无误。TCP协议使用重发机制——当一个通信实体发送一个消息给另一个通信实体后，需要收到另一个通信实体的确认信息，如果没有收到另一个通信实体的确认信息，则会再次重发刚才发送的信息。通过这种重发机制，TCP协议向应用程序提供了可靠的通信连接，使它能够自动适应网上的各种变化。即使在Internet暂时出现堵塞的情况下，TCP也能够保证通信的可靠性。

下图显示了TCP协议控制两个通信实体互相通信：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/tcp1.png)

综上所述，虽然IP和TCP这两个协议的功能不尽相同，也可以分开单独使用，但它们是在同一时期作为一个协议来设计的，并且在功能上也是互补的。只有两者结合起来，才能保证Internet在复杂的环境下正常运行。凡是要连接到Internet的计算机，都必须同时安装和使用这两个协议，因此在实际中常把这两个协议统称为TCP/IP协议。

## 二、Socket通信

### 2.1 使用ServerSocket创建TCP服务器端

在两个通信实体没有建立虚拟链路之前，必须有一个通信实体先做出“主动姿态”，主动接收来自其他通信实体的连接请求。Java中能接收其他通信实体连接请求的类是ServerSocket，ServerSocket对象用于监听来自客户端的Socket连接，如果没有连接，它将一直处于等待状态。ServerSocket包含一个监听来自客户端连接请求的方法。

~~~java
Socket accept():如果接收到一个客户端Socket的请求连接，改方法将返回一个与客户端Socket对应的Socket；否则该方法将一直处于等待状态，线程也被阻塞
~~~

为了创建ServerSocket对象，ServerSocket类提供了如下几个构造器：

~~~java
ServerSocket(int port)://用指定的端口port来创建一个ServerSocket。该端口应该有一个有效的端口整数值即0-65535
ServerSocket(int port,int backlog)://增加一个用来改变连接队列长度的参数backlog
ServerSocket(int port,int backlog,InetAddress localAddr)://在机器存在多个IP地址的情况下，允许通过localAddr参数来指定将ServerSocket绑定到指定的IP地址
~~~

当ServerSocket使用完毕后，应使用ServerSocket的close()方法来关闭该ServerSocket。

在通常情况下，服务器不应该只接收一个客户端请求，而应该不断地接收来自客户端的所有请求，所以Java程序通常会通过循环不断地调用ServerSocket的accept()方法。

~~~java
ServerSocket ss = new ServerSocket(30000);
while(true){
    //每接收到Socket请求时服务器端也产生一个Socket
    Socket s =ss.accept();
    //使用Socket通信
}
//上面程序中创建ServerSocket没有指定IP地址，则该ServerSocket将会绑定到本机默认的IP地址
~~~

### 2.2 使用Socket进行通信

客户端通常可以使用Socket的构造器来连接到指定服务器，Socket通常可以使用如下两个构造器。

~~~java
Socket(InetAddress/String remoteAddress,int port)://创建连接到指定远程主机、远程端口的Socket，该构造器没有指定本地地址、本地端口，默认使用本地主机的默认IP地址，默认使用系统动态分配的端口
Socket(InetAddress/String remoteAddress,int port，InetAddress localAddr,int localPort):://创建连接到指定远程主机、远程端口的Socket，并指定本地IP地址和本地端口，适用于本地主机有多个IP地址的情形。
~~~

上面两个构造器中指定远程主机时既可使用InetAddress来指定，也可直接使用String对象来指定，但程序通常使用String对象（如192.168.2.23）来指定远程IP地址。当本地主机只有一个IP地址时，使用第一个方法更为简单。

~~~java
Socket s = new Socket("127.0.0.1",30000);
~~~

当客户端、服务器端产生了对应的Socket之后，就得到了如上面的图片所示的通信示意图，程序无须再区分服务器端、客户端，而是通过各自的Socket进行通信。Socket提供了如下两个方法来获取输入流和输出流：

~~~java
InputStream getInputStream()://返回该Socket对象对应的输入流，让程序通过该输入流从Socket中取出数据。
OutputStream getOutputStream()://返回该Socket对象对应的输出流，让程序通过该输出流向Socket中输出数据。
~~~

例子：建立ServerSocket监听，并使用Socket获取输出流输出

~~~java
Server
public class ServerTest {
    public static void main(String[] args) throws IOException {
        //创建一个ServerSocket用于监听客户端Socket的连接请求
        ServerSocket ss = new ServerSocket(30000);
        while(true){
            Socket s = ss.accept();
            PrintStream ps = new PrintStream(s.getOutputStream());
            ps.println("恭喜你接收到了服务器的信息");
            ps.close();
            s.close();
        }
    }
}
Client
public class ClientTest {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1",30000);
        BufferedReader br = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        String line = br.readLine();
        System.out.println("来自服务器的数据："+line);
        br.close();
        socket.close();
    }
}
~~~

在实际应用中，程序可能不想让执行网络连接、读取服务器数据的进程一直阻塞，而是希望当网络连接、读取操作超过合理时间之后，系统自动认为该操作失败，这个合理时间就是超时时长。Socket对象提供了一个setSoTimeout(int timeout)方法来设置超时时长。如下代码片段所示：

~~~java
Socket s = new Socket("127.0.0.1",30000);
s.setSoTimeout(10000);
//让该SOcket连接到远程服务器，
s.connect(new InetAddress(host,port),10000);
~~~

当我们为Socket对象指定了超时时长之后，如果在使用Socket进行读、写操作完成之前超出了该时间限制，那么这些方法就会抛出SocketTimeoutException异常

### 2.3 加入多线程

实际应用中的客户端则可能需要和服务器端保持长时间通信，即服务器端需要不断地读取客户端数据，并向客户端写入数据；客户端也需要不断地读取服务器端数据，并向服务器端写入数据。当我们使用传统BufferedReader的readLine()方法读取数据时，在该方法成功返回之前，线程被阻塞，程序无法继续执行。考虑到这个原因，*服务器端应该为每个Socket单独启动一个线程*，每个线程负责与一个客户端进行通信。*客户端读取服务器端数据的线程同样会被阻塞*，所以系统应该单独启动一个线程，该线程专门负责读取服务器端数据。

现在考虑实现一个命令行界面的C/S聊天室应用，服务器端应该包含多个线程，每个Socket对应一个线程，该线程负责读取Socket对应输入流的数据（从客户端发送过来的数据），并将读到的数据向每个Socket输出流发送一次（将一个客户端发送的数据“广播”给其他客户端），因此需要在服务器端使用List来保存所有的Socket。

MyServer.java

~~~java
public class MyServer {
    public static ArrayList<Socket> socketList= new ArrayList<>();

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(30000);
        while (true){
            //下面这一行代码会一直阻塞等待别人的连接
            Socket socket = serverSocket.accept();
            socketList.add(socket);
            //每当客户端连接后启动一个ServerThread线程为该客户端服务
            new Thread(new ServerThread(socket)).start();
        }
    }
}
~~~

ServerThread.java

~~~java
public class ServerThread implements Runnable {

    Socket s = null;

    BufferedReader br = null;

    public ServerThread(Socket socket) throws IOException {
        this.s =socket;
        br = new BufferedReader(new InputStreamReader(s.getInputStream()));
    }

    private String readFromClient(){
        try {
            return br.readLine();
        } catch (IOException e) {
            //如果捕获到异常，则表明该Socket对应的客户端已经关闭
            MyServer.socketList.remove(s);
            e.printStackTrace();
        }
        return null;
    }

    @Override
    public void run() {
        try {
            String content = null;
            while (( content=readFromClient() ) != null){
                for (Socket sk:MyServer.socketList) {
                    PrintStream ps =new PrintStream(sk.getOutputStream());
                    ps.println(content);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
~~~

MyClient.java

~~~java
public class MyClient {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1",30000);
        //客户端启动ClientThread线程不断地读取来自服务器的数据
        new Thread(new ClientThread(socket)).start();
        //获取该Socket对用的输出流
        PrintStream ps = new PrintStream(socket.getOutputStream());
        String line = null;
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        while ((line=br.readLine())!= null){
            //将用户的键盘输入内容写入Socket对应的输出流
            ps.println(line);
        }
    }
}
~~~

ClientThread.java

~~~java
public class ClientThread implements Runnable {
	//该线程负责处理的Socket
    private Socket socket;
	//该线程处理的Socket输入流
    BufferedReader br = null;

    public ClientThread(Socket socket) throws IOException {
        this.socket=socket;
        br= new BufferedReader(new InputStreamReader(socket.getInputStream()));
    }

    @Override
    public void run() {
        try {
            String content = null;
            while ((content=br.readLine())!=null){
                System.out.println(content);
            }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
~~~

### 2.4 记录用户信息

上面程序虽然已经完成了粗略的通信功能，每个客户端可以看到其他客户端发送的信息，但无法知道是哪个客户端发送的信息，这是因为服务器端从未记录过用户信息，当客户端使用Socket连接到服务器端之后，程序只是使用socketList集合保存了服务器端对应生成的Socket，并没有保存该Socket关联的客户信息。

下面程序将考虑使用Map来保存用户状态信息，因为本程序将考虑实现私聊功能，也就是说，**一个客户端可以将信息发送给另一个指定客户端**。实际上所有的客户端只与服务器端连接，客户端之间并没有互相连接，也就是说，当一个客户端信息发送到服务器端之后，服务器端必须可以判断该信息到底是向所有用户发送，还是向指定用户发送，并需要知道向哪个用户发送。这里需要解决如下两个问题：

1. 客户端发送来的信息必须有特殊的标识——让服务器端可以判断是公聊信息，还是私聊信息。
2. 如果是私聊信息，客户端会发送该消息的目的用户（私聊对象）给服务器端，服务器端如何将该信息发送给该私聊对象。

为了解决第一个问题，我们可以让客户端在发送不同信息之前，先对这些信息进行适当处理，比如在内容前后添加一些特殊字符——我们把这种特殊字符称为协议字符。本例提供了一个CrazyitProtocol接口，该接口专门用于定义协议字符。

CrazyitProtocol.java

~~~java
public interface CrazyitProtocol
{
    //定义协议字符串的长度
    int PROTOCOL_LEN = 2;
    //下面是一些协议字符串，服务器和客户端交换的信息
    //都应该在前、后添加这种特殊字符串。
    String MSG_ROUND = "§γ";
    String USER_ROUND = "∏∑";
    String LOGIN_SUCCESS = "1";
    String NAME_REP = "-1";
    String PRIVATE_ROUND = "★【";
    String SPLIT_SIGN = "※";
}
~~~

为了解决第二个问题，我们可以考虑使用一个Map来保存聊天室所有用户和对应Socket之间的映射关系——这样服务器端就可以根据用户名来找到对应的Socket。

但实际上本程序并未这么做，程序仅仅是用Map保存了聊天室所有用户名和对应输出流之间的映射关系，因为服务器端只要获取该用户名对应的输出流即可。服务器端提供了一个HashMap的子类，该类不允许value重复，并提供了根据value获取key，根据value删除key等方法。

CrazyitMap.java

~~~java
package com.socket;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Set;

//拓展HashMap类，CrazyitMap类要求value也不可重复
public class CrazyitMap<K,V> extends HashMap<K,V>{
    //跟据value来删除指定项
    public void removeByValue(Object value){
        for(Object key: keySet()){
           if (get(key)==value){
               remove(key);
               break;
           }
        }
    }
    
    //获取所有value组成的set集
    public Set<V> valueSet(){
        Set<V> result = new HashSet<V>();
        for (K key:keySet()) {
            result.add(get(key));
        }
        return result;
    }
    
    //跟据value查找key
    public K getKeyByValue(V value){
        for (K key:keySet()) {
            //如果指定key对应的value与被搜索的相同则返回对应的key
            if (get(key).equals(value)&&get(key)==value){
                return key;
            }
        }
        return null;
    }
    
    //重写hashMap的put方法，不允许value重复
    public V put(K key,V value){
        for (V val:valueSet()) {
            //如果指定value与试图放入集合的value相同，则抛出一个异常
            if (val.equals(value)&&val.hashCode()==value.hashCode()){
                throw new RuntimeException("不允许重复value");
            }
        }
        return super.put(key,value);
    }
}

~~~

服务器端的主类一样只是建立ServerSocket来监听来自客户端Socket的连接请求，但该程序增加了一些异常处理，可能看上去比上一节的程序稍微复杂一点。

Server.java

~~~java
public class Server {
    private static final int SERVER_PORT=30000;

    //使用CrazyitMap对象来保存每个客户端名字和对应输出流的对应关系
    public static CrazyitMap<String,PrintStream> clients = new CrazyitMap<>();

    public void init(){
        try {
            ServerSocket serverSocket = new ServerSocket(SERVER_PORT);
            while (true){
                //下面这一行代码会一直阻塞等待别人的连接
                Socket socket = serverSocket.accept();
                //每当客户端连接后启动一个ServerThread线程为该客户端服务
                new Thread(new ServerThread(socket)).start();
            }

        }catch (Exception e){
            System.out.println("服务器启动失败，是否端口"+SERVER_PORT+"已被占用");
        }
    }

    public static void main(String[] args) throws IOException {
        Server server =new Server();
        server.init();

    }
}
~~~

ServerThread.java

~~~java
public class ServerThread implements Runnable {

    private Socket socket;

    BufferedReader br = null;

    PrintStream ps =null;

    public ServerThread(Socket socket) throws IOException {
        this.socket =socket;
    }


    @Override
    public void run() {
        try {
            br=new BufferedReader(new InputStreamReader(socket.getInputStream()));
            ps =new PrintStream(socket.getOutputStream());
            String line = null;
            while (( line=br.readLine()) != null){
                //如果读到的行以CrazyProtocol.USER_ROUND开始，并以其结束，
                //则可以确定独到的是用户登录的用户名
                if (line.startsWith(CrazyitProtocol.USER_ROUND)&&line.endsWith(CrazyitProtocol.USER_ROUND)){
                    //得到真实消息
                    String userName = getRealMsg(line);
                    //如果用户名重复
                    if (Server.clients.containsKey(userName)){
                        System.out.println("重复");
                        ps.println(CrazyitProtocol.NAME_REP);
                    }else {
                        System.out.println("成功");
                        ps.println(CrazyitProtocol.LOGIN_SUCCESS);
                        Server.clients.put(userName,ps);
                    }
                }
                //如果读到的行以CrazyProtocol.PRIVATE_ROUND开始，并以其结束，
                //则可以确定是私聊信息，私聊信息只向特定的输出流发送
                else if(line.startsWith(CrazyitProtocol.PRIVATE_ROUND)&&line.endsWith(CrazyitProtocol.PRIVATE_ROUND)){
                    String userAndMsg=getRealMsg(line);
                    //以SPLIT_SIGN分割字符串，前半是私聊用户，后半是聊天信息
                    String user = userAndMsg.split(CrazyitProtocol.SPLIT_SIGN)[0];
                    String msg = userAndMsg.split(CrazyitProtocol.SPLIT_SIGN)[1];
                    //获取私聊用户对应的输出流，并发送私聊信息
                    Server.clients.get(user).println(Server.clients.getKeyByValue(ps)+"悄悄对你说："+msg);
                }
                //公聊要向每个Socket发送
                else {
                    String msg =getRealMsg(line);
                    //遍历clients中的每个输出流
                    for (PrintStream cps:Server.clients.valueSet()) {
                        cps.println(Server.clients.getKeyByValue(ps)+"说："+msg);
                    }
                }
            }
            //捕获到异常后，表明Socket对应的客户端已经出现了问题
            //所以程序将其对应的输出流从Map中删除
        } catch (IOException e) {
            Server.clients.removeByValue(ps);
            System.out.println(Server.clients.size());
        }

        try {
            if (br!=null){
                br.close();
            }
            if (ps!=null){
                ps.close();
            }if (socket!=null){
                socket.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private String getRealMsg(String line) {
        return line.substring(CrazyitProtocol.PROTOCOL_LEN,line.length()-CrazyitProtocol.PROTOCOL_LEN);
    }

}
~~~

Client.java

~~~java
public class Client {

    private static final int SERVER_PORT=30000;

    private Socket socket;

    private PrintStream ps;

    private BufferedReader brServer;

    private BufferedReader keyIn;

    public void init(){
        try{
            //初始化代表键盘的输入流
            keyIn=new BufferedReader(new InputStreamReader(System.in));
            //连接到服务器端
            socket=new Socket("127.0.0.1",SERVER_PORT);
            ps=new PrintStream(socket.getOutputStream());
            brServer = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            String tip="";
            //采用循环不断地弹出对话框要求输入用户名
            while (true){
                String userName = JOptionPane.showInputDialog(tip+"输入用户名");
                //在用户输入的用户名前后增加协议字符串
                ps.println(CrazyitProtocol.USER_ROUND+userName+CrazyitProtocol.USER_ROUND);
                //读取服务器端的响应
                String result = brServer.readLine();
                //如果用户名重复，则开始下一次循环
                if (result.equals(CrazyitProtocol.NAME_REP)){
                    tip="用户名重复！重新输入";
                    continue;
                }
                //如果服务器端返回登陆成功，则循环结束
                if (result.equals(CrazyitProtocol.LOGIN_SUCCESS)){
                    break;
                }
            }
        } catch (UnknownHostException e) {
            System.out.println("找不到远程服务器，请确定服务器已经启动");
            closeRs();
            System.exit(1);
        }catch (IOException e){
            System.out.println("网络异常，请重新登录");
            closeRs();
            System.exit(1);
        }
        //以Socket对应的输入流启动ClientThread线程

    }
    public static void main(String[] args) throws IOException {

    }

    private void readAndSend(){
        try{
            String line = null;
            while ((line=keyIn.readLine()) !=null){
                //如果发送的信息中有冒号，且以/开头，则认为想发送私聊信息
                if (line.indexOf(":")>0&&line.startsWith("/")){
                    line=line.substring(1);
                    ps.println(CrazyitProtocol.PRIVATE_ROUND+line.split(":")[0]+CrazyitProtocol.PRIVATE_ROUND);
                }else {
                    ps.println(CrazyitProtocol.MSG_ROUND+line+CrazyitProtocol.MSG_ROUND);
                }
            }
        } catch (IOException e) {
            System.out.println("请重新登录");
            closeRs();
            e.printStackTrace();
        }
    }

    private void closeRs(){
        try{
            if (keyIn != null){
                ps.close();
            }
            if (brServer!= null){
                ps.close();
            }
            if (ps!= null){
                ps.close();
            }
            if (socket!= null){
                keyIn.close();
            }
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}

~~~

ClientThread.java

~~~java
public class ClientThread implements Runnable {
    //该线程处理的Socket输入流
    BufferedReader br = null;

    public ClientThread(BufferedReader br) throws IOException {
        this.br=br;
    }

    @Override
    public void run() {
        try {
            String content = null;
            while ((content=br.readLine())!=null){
                System.out.println(content);
                /*  本例仅打印了从服务器端独到的内容。实际上，可以更复杂
                *   如果我们希望客户端能看到聊天室的用户列表，则可以让服务器端在每次由用户登录、用户退出时
                *   将所有的用户列表信息都向客户端发送一遍，为了区分是聊天信息还是用户列表，
                *   服务器端也应该添加一定的协议字符串。
                * */
            }

        }catch (Exception e){
            e.printStackTrace();
        }finally {
            try {
                if (br!=null){
                    br.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

~~~

## 三、使用NIO实现非阻塞Socket通信

从JDK 1.4开始，Java提供了NIO API来开发高性能的网络服务器，前面介绍的网络通信程序是基于阻塞式API的——即当程序执行输入、输出操作后，在这些操作返回之前会一直阻塞该线程，所以服务器端必须为每个客户端都提供一个独立线程进行处理，当服务器端需要同时处理大量客户端时，这种做法会导致性能下降。使用NIO API则可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。

# NIO编程

## 一、简介

java.nio全称java non-blocking IO，是指jdk提供的新的API，从1.4开始java提供了一系列改进输入输出的新特性，被统称为NIO（New IO）。新增了许多用于处理输入输出的类，这些类都放在java.nio包及子包下，并且对原java.io包中的很多类进行改写，新增了满足NIO的功能

NIO和BIO有着相同的目的和作用，但它们的实现方式完全不同，BIO是以流的方式处理数据，而NIO以块的方式处理，处理效率高很多。

NIO有三大核心部分，Channel（通道），Buffer（缓冲区），Selector（选择器）。传统的BIO是基于字节流和字符流进行操作，而NIO基于CHannel和Buffer进行操作，数据综述从通道读取到缓冲区，或者从缓冲区写入到通道正。selector用于监听多个通道的时间，因此单个线程就可以监听多个客户端通道。

## 二、文件IO

### 2.1 概述和核心api

**buffer**：实际上就是一个容器，是一个特殊的数组，缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。Channel提供从文件、网络读取数据的渠道，但是读取和写入的数据必须经过buffer.

在nio中，buffer是一个顶层父类，他是一个抽象类，常用的buffer子类有：

- ByteBuffer，存储字节数据到缓冲区
- ShortBuffer，存储短整型数据到缓冲区
- CharBuffer，存储字符数据到缓冲区
- IntBuffer，存储整形数据到缓冲区
- LongBuffer，存储长整形数据到缓冲区
- ......

对于Java中的基本数据类型，都有一个BUffer类型与之对应。上面这些Buffer类，除了ByteBuffer之外，它们都采用相同或相似的方法来管理数据，只是各自管理的数据类型不同而已。这些Buffer类都没有提供构造器，通过使用如下方法来得到一个Buffer对象：

~~~java
static XxxBuffer allocate(int capacity):
//创建一个容量为capacity的XxxBuffer对象
~~~

在Buffer中有三个概念：容量（capacity）、界限（limit）和位置（position）

> 容量：缓冲区的容量表示该Buffer的最大数据容量，缓冲区的容量不能为负，创建后不能改变。
>
> 界限：第一个不应该被读出或写入的缓冲区位置索引。也就是说，位于limit后的数据既不可被读也不可被写
>
> 位置：用于指明下一个可以被读出的或写入的缓冲区位置索引。当使用Buffer冲Channel中读取数据时，position的值刚好等于已经读到了多少数据。

开始时Buffer的position为0，limit为capacity，程序可通过put()方法向Buffer中放入一些数据（或者从Channel中获取一些数据），每放入一些数据，Buffer的position相应地向后移动一些位置。

当Buffer装入数据结束后，调用Buffer的flip()方法，该方法将limit设置为position所在位置，并将position设为0，这就使得Buffer的读写指针又移到了开始位置。也就是说，Buffer调用flip()方法之后，Buffer为输出数据做好准备；当Buffer输出数据结束后，Buffer调用clear()方法，clear()方法不是清空Buffer的数据，它仅仅将position置为0，将limit置为capacity，这样为再次向Buffer中装入数据做好准备。

除了flip()和clear()之外Buffer还包含以下一些方法：

~~~java
int capacity()：返回Buffer的capacity大小。
boolean hasRemaining()：判断当前位置（position）和界限（limit）之间是否还有元素可供处理。
int limit()：返回Buffer的界限（limit）的位置。
Buffer limit(int newLt)：重新设置界限（limit）的值，并返回一个具有新的limit的缓冲区对象。
Buffer mark()：设置Buffer的mark位置，它只能在0和位置（position）之间做mark。
int position()：返回Buffer中的position值。
Buffer position(int newPs)：设置Buffer的position，并返回position被修改后的Buffer对象。
int remaining()：返回当前位置和界限（limit）之间的元素个数。
Buffer reset()：将位置（position）转到mark所在的位置。
Buffer rewind()：将位置（position）设置成0，取消设置的mark。
~~~

Buffer的所有子类还提供了两个重要的方法：put()和get()方法，用于向Buffer中放入数据和从Buffer中取出数据。当使用put()和get()方法放入、取出数据时，Buffer既支持对单个数据的访问，也支持对批量数据的访问（以数组作为参数）。

**Channel**：类似于BIO的stream。例如FileInputStream对象，用来建立到目标（文件，网络套接字，硬件设备）的一个连接，但是BIO是单向的，而NIO是双向的，既可以用来读也可以写。常见的Channel类有：FileChannel（用于文件读写）、DatagramChannel（用于UDP的读写）、ServerSocketChannel、SocketChannel（用于TCP的数据读写）。在文件IO中只介绍FileChannel。

FileChannel类主要用来对本地文件进行IO操作主要方法如下：

- public int read(ByteBuffer dst),从通道读取数据并放到缓冲区
- public int write(ByteBuffer src)，把缓冲区的数据写入通道中
- public long transferFrom(ReadableByteChannel src,long position,long count)，从目标通道中复制数据到当前通道
- public long transferTo(long position,long count,WritableByteChannel target)，把数据从当前通道复制给目标通道

### 2.2 案例

**向本地文件中写数据**

~~~java
    @Test
    public void test1() throws Exception{
        String str = "hello,nio,i am Loserfromlazy";
        FileOutputStream fileOutputStream = new FileOutputStream("E://obj.txt");
        FileChannel fileChannel = fileOutputStream.getChannel();
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        byteBuffer.put(str.getBytes());
        byteBuffer.flip();
        fileChannel.write(byteBuffer);
        fileOutputStream.close();
    }
~~~

NIO的通道是从输入流对象里通过getChannel方法获取的，该通道是双向的，既可以读又可以写。在网通道里写入数据之前，必须通过put方法把数据存入ByteBuffer中，然后通过通道的write方法写数据。在write之前需要调用flip方法反转缓冲区，把重置到初始位置，这样接下来写数据时才能把所有数据写到通道里

**从本地文件中读数据**

~~~java
    @Test
    public void test2() throws Exception{
        File file = new File("E://obj.txt");
        FileInputStream inputStream = new FileInputStream(file);
        FileChannel fileChannel = inputStream.getChannel();
        ByteBuffer buffer = ByteBuffer.allocate((int) file.length());
        fileChannel.read(buffer);
        System.out.println(new String(buffer.array()));
        inputStream.close();
    }
~~~

**复制文件**

通过BIO

~~~java
    @Test
    public void test3() throws Exception{
        FileInputStream is = new FileInputStream("E://temp/daddy.mp4");
        FileOutputStream os = new FileOutputStream("E://temp/da/daddy.mp4");
        byte []  bytes = new byte[1024];
        while (true){
            int res = is.read(bytes);
            if(res == -1){
                break;
            }
            os.write(bytes,0,res);
        }
        is.close();
        os.close();
    }
~~~

通过NIO

~~~java
    @Test
    public void test4() throws Exception{
        FileInputStream is = new FileInputStream("E://temp/daddy.mp4");
        FileOutputStream os = new FileOutputStream("E://temp/da/daddy.mp4");
        FileChannel sourceFC = is.getChannel();
        FileChannel destFC = os.getChannel();
        destFC.transferFrom(sourceFC,0,sourceFC.size());
        sourceFC.close();
        destFC.close();
    }
~~~

## 三、网络IO

### 3.1 概述

上一部分文件IO用到的FileChannel并不支持非阻塞IO，NIO主要还是为了网络IO。java nio 的网络通道是非阻塞IO的实现，基于事件驱动。

在Java中编写Socket服务器，通常有以下几种模式：

- 一个客户端连接用一个线程，优点：程序编写简单。缺点：如果连接非常多，分配的线程也会非常多，服务器可能会因为资源耗尽而崩溃。
- 把每一个客户端连接交给一个拥有固定数量线程的连接池，优点：程序编写相对简单，可以处理大量的连接。缺点：线程的开销非常大，连接如果非常多，排队现象会比较严重。
- 使用Java的NIO，用非阻塞式IO方式处理。这种模式可以用一个线程，处理大量的客户端连接。

Java的NIO为非阻塞式Socket通信提供了如下几个特殊类。

**Selector**：能够检测多个注册的通道上是否有事件发生，如果有事件发生，便获取事件然后对每个进行相应的处理。这样就可以只用一个单线程去管理多个通道，也就是管理多个连接。这样使得只有在连接真正有读写事件发生时，才会调用函数来进行读写，就大大减少了系统开销，并且不必为每个连接都创建一个线程，不用维护多个线程，并且避免了多线程之间的上下文切换导致的开销。

该类的常用方法：

~~~java
public static Selector open()//得到一个选择器对象
public int select(long timeout)//监控所有注册的通道，当其中有IO操作可以进行时，将对应的SelectionKey加入到内部集合中并返回，参数用来设置超过时间
public Set<SelectionKey>selectedKeys()//从内部集合中得到所有的SelectionKey
~~~

**SelectionKey**：代表了Selector和网络通道的注册关系，一共四种：

- int OP_ACCEPT:有新的网络连接可以accept，值为16
- int OP_CONNECT:代表连接已经建立，值为8
- int OP_READ 和 int OP_WRITE:代表了读、写操作，值为1和4

该类的常用方法如下：

~~~java
public abstract Selector selector()//得到与之关联的通道
public abstract SelectableChannel channel()//得到与之关联的通道
public final Object attachment()//得到与之关联的共享数据
public abstract SelectionKey interestOps(int ops)//设置或改变监听事件
public final boolean isAcceptable()//是否可以accept
public final boolean isReadable()//是否可以读
public final boolean isWritable()//是否可以写
~~~

**ServerSocketChannel**：用来在服务器端监听新的客户端Socket连接，常用方法如下：

~~~java
public static ServerSocketChannel open()//得到一个ServerSocketChannel通道
public final ServerSocketChannel bind()//设置服务器端口号
public final SelectableChannel configureBlocking(bollean block)//设置阻塞或非阻塞模式，取值false表示采用非阻塞模式
public SocketChannel accept()//接受一个链接并返回这个连接的通道对象
public fianl SelectionKey register(Selector sel,int ops)//注册一个选择器并设置监听事件
~~~

**SocketChannel**:网络IO通道，具体负责进行读写操作。NIO总是把缓冲区的数据写入通道，或者把通道里的数据读到缓冲区，常用方法如下：

~~~java
public static SocketChannel open()//得到一个SocketChannel通道
public final SelectableChannel configureBlocking(bollean block)//设置阻塞或者非阻塞模式，取值false表示采用非阻塞模式
public boolean connect(SocketAddress remote)//连接服务器
public boolean finishConnect()//如果上面的方法连接失败，则通过改方法完成连接操作
public int write(ByteBuffer src)//往通道里写数据
public int read(ByteBuffer src)//从通道里读数据
public final SelectionKey register(Selector sel,int ops,Object att)//注册一个选择器并设置监听事件，最后一个参数可以设置共享数据
public final void close()//关闭通道
~~~

### 3.2 案例

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/nio1.png)

**入门案例——非阻塞实现服务器端和客户端之间的数据通信**

NIOServer.java

~~~java
public class NioServer {

    public static void main(String[] args) throws IOException {
        //1.得到一个ServerSocketChannel对象
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        //2.得到一个Selector对象
        Selector selector = Selector.open();
        //3.绑定一个端口号
        serverSocketChannel.bind(new InetSocketAddress(9999));
        //4.设置非阻塞方式
        serverSocketChannel.configureBlocking(false);
        //5. 把ServerSocketChannel对象注册给Selector对象
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        //6. 开始
        while(true){
            // 6.1 监控客户端
            if (selector.select(2000)==0){
                System.out.println("Server:没有客户端搭理我，我干别的活");
                continue;
            }
            //6.2 得到SelectionKey，判断通道里的事件
            Iterator <SelectionKey> keyiterator = selector.selectedKeys().iterator();
            while (keyiterator.hasNext()){
                SelectionKey key = keyiterator.next();
                if (key.isAcceptable()){//客户端连接请求事件
                    System.out.println("OP_ACCEPT");
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector,SelectionKey.OP_READ, ByteBuffer.allocate(1024));
                }
                if (key.isReadable()) {//读取客户端事件
                    SocketChannel channel = (SocketChannel) key.channel();
                    ByteBuffer buffer = (ByteBuffer) key.attachment();
                    channel.read(buffer);
                    System.out.println("客户端发来数据："+new String(buffer.array()));
                }
                //6.3 从集合中移除当前key防止重复处理
                keyiterator.remove();
            }
        }
    }
}
~~~

NIOClient.java

~~~java
public class NioClient {
    public static void main(String[] args) throws IOException {
        //1.得到一个网络通道
        SocketChannel channel =SocketChannel.open();
        //2.设置非阻塞方式
        channel.configureBlocking(false);
        //3. 提供服务器端的IP地址和端口号
        InetSocketAddress address = new InetSocketAddress("127.0.0.1",9998);
        //4. 连接服务器端
        if (!channel.connect(address)){
            while (!channel.finishConnect()){//nio非阻塞的优势
                System.out.println("Client: 链接服务器的同时，干别的事");
            }
        }
        //5. 得到一个缓冲区并存入数据
        String msg = "hello Server";
        ByteBuffer writebuffer = ByteBuffer.wrap(msg.getBytes());
        //6. 发送数据
        channel.write(writebuffer);
        System.in.read();
    }
}
~~~

**网络聊天案例**

ChatServer.java

~~~java

~~~

ChatClient.java

~~~java

~~~

TestChat.java

~~~java

~~~

### 3.3 IO对比总结

IO通常分为几种，同步阻塞的BIO、同步非阻塞的NIO，异步非阻塞的AIO

- BIO适用于连接数目较小且固定的结构，这种方式对服务器资源要求较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序直观简单易理解
- NIO适用于连接数目较多且比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂
- AIO适用于连接数较多且连接比较长的架构，比如相册服务器，充分调用OS参与并发操作，编程比较复杂，JDK1.7开始支持

# Netty学习

## 一、简介

### 1.1 netty

Netty是由JBOSS提供的一个java开源框架，现为 Github上的独立项目。Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。

也就是说，Netty 是一个基于NIO的客户、服务器端的编程框架，使用Netty 可以确保你快速和简单的开发出一个网络应用，例如实现了某种协议的客户、服务端应用。Netty相当于简化和流线化了网络应用的编程开发过程，例如：基于TCP和UDP的socket服务开发。

“快速”和“简单”并不用产生维护性或性能上的问题。Netty 是一个吸收了多种协议（包括FTP、SMTP、HTTP等各种二进制文本协议）的实现经验，并经过相当精心设计的项目。最终，Netty 成功的找到了一种方式，在保证易于开发的同时还保证了其应用的性能，稳定性和伸缩性。 [1]

应用场景：

JavaEE：dubbo

### 1.2 BIO、NIO、AIO

**阻塞与非阻塞**

主要指访问io的线程是否会阻塞

主要是线程访问资源，该资源是否准备就绪的一种处理方式

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty1.png)

**同步和异步**

主要指的是数据的请求方式，同步和异步是指访问数据的一种机制



![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty2.png)

**BIO**

同步阻塞IO，Block IO，IO操作时会阻塞线程，并发处理能力低

Socket编程就是BIO，一个socket连接一个处理线程，阻塞的原因在于，操作系统允许的线程数量是有限的，多个socket申请与服务端建立连接时，服务端不能提供相应数量的处理线程，没有分配到处理线程的连接就会阻塞等待或被拒绝

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty3.png)

**NIO**

同步非阻塞IO，None-Block IO

NIO是对BIO的改进，基于Reactor模型。我们知道，一个socket连接只有在特点时候才会发生数据传输IO操作，大部分时间这个“数据通道”是空闲的，但还是占用着线程。NIO作出的改进就是“一个请求一个线程”，在连接到服务端的众多socket中，只有需要进行IO操作的才能获取服务端的处理线程进行IO。这样就不会因为线程不够用而限制了socket的接入。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty4.png)

**AIO**

*异步非阻塞IO*

这种IO模型是由操作系统先完成了客户端请求处理再通知服务器去启动线程进行处理。AIO也称NIO2.0，在JDK7开始支持。

### 1.3 NettyReactor模型 

单线程模型、多线程模型、主从多线程模型

#### 1.3.1 单线程模型

用户发起IO请求到Reactor线程

Ractor线程将用户的IO请求放入到通道，然后再进行后续处理

处理完成后，Reactor线程重新获得控制权，继续其他客户端的处理

> 这种模型一个时间点只有一个任务在执行，这个任务执行完了，再去执行下一个任务。
>
> *1.*    *但单线程的**Reactor**模型每一个用户事件都在一个线程中执行：*
>
> *2.*    *性能有极限，不能处理成百上千的事件*
>
> *3.*    *当负荷达到一定程度时，性能将会下降*
>
> *4.*    *某一个事件处理器发生故障，不能继续处理其他事件*

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty5.png)

#### 1.3.2 多线程模型

Reactor多线程模型是由一组NIO线程来处理IO操作（之前是单个线程），所以在请求处理上会比上一中模型效率更高，可以处理更多的客户端请求。

> *这种模式使用多个线程执行多个任务，任务可以同时执行*
>
> 但是如果并发仍然很大，Reactor仍然无法处理大量的客户端请求

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty6.png)

#### 1.3.3 主从多线程模型

这种线程模型是Netty推荐使用的线程模型

这种模型适用于高并发场景，一组线程池接收请求，一组线程池处理IO。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty7.png)































