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

“快速”和“简单”并不用产生维护性或性能上的问题。Netty 是一个吸收了多种协议（包括FTP、SMTP、HTTP等各种二进制文本协议）的实现经验，并经过相当精心设计的项目。最终，Netty 成功的找到了一种方式，在保证易于开发的同时还保证了其应用的性能，稳定性和伸缩性。 

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

用户发起IO请求到Reactor线程，Ractor线程将用户的IO请求放入到通道，然后再进行后续处理，处理完成后，Reactor线程重新获得控制权，继续其他客户端的处理

服务器端用一个线程通过多路复用搞定所有的IO操作，包括读写连接

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

服务器端采用一个线程专门处理客户端连接请求，采用一个线程池负责 IO 操作。在绝大多数场景下，该模型都能满足使用。

> *这种模式使用多个线程执行多个任务，任务可以同时执行*
>
> 但是如果并发仍然很大，Reactor仍然无法处理大量的客户端请求

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty6.png)

#### 1.3.3 主从多线程模型

这种线程模型是Netty推荐使用的线程模型

这种模型适用于高并发场景，一组线程池接收请求，一组线程池处理IO。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty7.png)

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty8.png)

类似于上面的线程池模型，Netty 抽象出两组线程池，BossGroup 专门负责接收客户端连接，WorkerGroup 专门负责网络读写操作。NioEventLoop 表示一个不断循环执行处理任务的线程，每个 NioEventLoop 都有一个 selector，用于监听绑定在其上的 socket 网络通道。NioEventLoop 内部采用串行化设计，从消息的读取->解码->处理->编码->发送，始终由 IO 线程 NioEventLoop 负责

一个NioEventLoopGroup包含多个NioEventLoop

每个NioEventLoop包含一个Selector，一个taskQueue

每个NioEventLoop的Selector上可以注册多个NioChannel

每一个NioChannel都会绑定唯一的NioEventLoop上

每个NioChannel都绑定一个自己的ChannelPipeline

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty9.png)

BossGroup 线程维护 Selector，只关注 Accecpt
当接收到 Accept 事件，获取到对应的 SocketChannel，封装成 NIOScoketChannel 并注册到 Worker 线程（事件循环），并进行维护
当 Worker 线程监听到 Selector 中通道发生自己感兴趣的事件后，就进行处理（就由 handler），注意 handler 已经加入到通道

### 1.4 入门案例

服务器端业务处理类

```java
public class NettyServerHandler extends ChannelInboundHandlerAdapter {
    //读取数据事件
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("Server:"+ctx);
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("客户端发来的消息："+buf.toString(CharsetUtil.UTF_8));
    }
    //数据读取完毕事件
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.copiedBuffer("就是没钱",CharsetUtil.UTF_8));
    }
    //异常发生事件
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

服务器类

```java
public class NettyServer {
    public static void main(String[] args) throws InterruptedException {
        //1.创建线程组
        //创建BossGroup 和 WorkerGroup
        //说明
        //1. 创建两个线程组 bossGroup 和 workerGroup
        //2. bossGroup 只是处理连接请求 , 真正的和客户端业务处理，会交给 workerGroup完成
        //3. 两个都是无限循环
        //4. bossGroup 和 workerGroup 含有的子线程(NioEventLoop)的个数
        //   默认实际 cpu核数 * 2
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        //2.创建服务器端启动助手来配置参数
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup,workerGroup)//设置线程组
                .channel(NioServerSocketChannel.class)//使用NioServerSocketChannel作为服务器端通道实现
                .option(ChannelOption.SO_BACKLOG,128)//设置线程队列中等待连接的个数
                .childOption(ChannelOption.SO_KEEPALIVE,true)//保持活动连接状态
                 //handler对应 bossGroup , childHandler 对应 workerGroup
                .childHandler(new ChannelInitializer<SocketChannel>() {//创建一个通道初始化对象
                    //往Pipeline链中添加自定义业务处理handler
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new NettyServerHandler());
                        System.out.println("========Server is Ready========");
                    }
                });
        //3.启动服务器端并绑定端口，等待客户端连接（非阻塞）
        //绑定一个端口并且同步生成了一个 ChannelFuture 对象（也就是立马返回这样一个对象）
        ChannelFuture channelFuture = bootstrap.bind(9999).sync();
        System.out.println("========Server is Start========");
        //4.关闭通道，关闭线程池
        //对关闭通道事件  进行监听
        channelFuture.channel().closeFuture().sync();
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }

}
```

客户端业务处理类

```java
public class NettyClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("Client:"+ctx);
        ctx.writeAndFlush(Unpooled.copiedBuffer("老板，还钱",CharsetUtil.UTF_8));
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        System.out.println("服务器端发来的消息："+in.toString(CharsetUtil.UTF_8));
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

客户端类

```java
public class NettyClient {
    public static void main(String[] args) throws InterruptedException {
        //创建一个EventLoopGroup线程组
        //客户端需要一个事件循环组
        EventLoopGroup group =new NioEventLoopGroup();
        //创建客户端启动助手
        //注意客户端使用的不是 ServerBootstrap 而是 Bootstrap
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group).channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new NettyClientHandler());
                        System.out.println("========Client is ready=========");
                    }
                });
        ChannelFuture channelFuture = bootstrap.connect("127.0.0.1",9999).sync();
        channelFuture.channel().closeFuture().sync();
    }
}
```

### 1.5 编码与解码Protobuf

在编写网络应用程序的事后需要注意codec（编解码器），因为数据在网络中传输时是二进制字节码数据，而我们的目标数据不是字节码数据，因此发送数据时需要编码接收数据时需要解码。codec组成部分有两个：decoder（解码器）和encoder（编码器）。encoder把业务数据转换成字节码数据，decoder把字节码数据转换成业务数据。

java的序列化技术也可以作为codec使用，但有很多缺点：

1. 无法跨语言
2. 序列化后体积过大，是二进制的5倍
3. 性能低

netty自身也有一些codec：编码器：StringDecoder、ObjectDecoder...解码器：StringEncoder、ObjectEncoder...

Netty 本身自带的 ObjectDecoder 和 ObjectEncoder 可以用来实现 POJO 对象或各种业务对象的编码和解码，但其内部使用的仍是 Java 序列化技术。因此对于 POJO 对象或各种业务对象要实现编码和解码使用Google的Protobuf。

Protobuf时Google的开源项目，支持跨平台多语言，高性能和高可靠性等特点。

**使用方法：**

1 导入依赖

~~~xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.11.4</version>
</dependency>
~~~

2 编写proto文件

```protobuf
syntax="proto3";//设置版本号，3为大版本号即3.11.4中的3
option java_outer_classname = "BookMessage";//设置生成的Java类名
message Book{
  int32 id =1;//设置类中的属性，等号后是序号不是属性值
  string name =2;
}
```

3 去官网下载同版本的protoc.exe

在命令行窗口中使用

进入到protoc.exe所在的目录，将proto文件也放在该目录下，使用如下命令

~~~shell
protoc --java_out=. Book.proto
~~~

将生成的.java文件拷贝到项目中打开（直接用即可，不要编辑）在netty中使用

4 在netty中使用

在入门案例中进行修改：

NettyClient.java

```java
ch.pipeline().addLast("encoder",new ProtobufEncoder());
ch.pipeline().addLast(new NettyClientHandler());
System.out.println("========Client is ready=========");
```

NettyClientHandler.java

```java
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    BookMessage.Book book =BookMessage.Book.newBuilder().setId(1).setName("这是一本好书").build();
    System.out.println("Client:"+ctx);
    ctx.writeAndFlush(book);
}
```

NettyServer.java

```java
ch.pipeline().addLast("decoder",new ProtobufDecoder(BookMessage.Book.getDefaultInstance()));
ch.pipeline().addLast(new NettyServerHandler());
System.out.println("========Server is Ready========");
```

NettyServerHandler.java

```java
//读取数据事件
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    System.out.println("Server:"+ctx);
    BookMessage.Book book = (BookMessage.Book) msg;
    System.out.println("客户端发来的消息："+book.getName());
}
```

## 二、核心组件

### 2.1 BootStrap、ServerBootStrap

Bootstrap 意思是引导，一个 Netty应用通常由一个 Bootstrap开始，主要作用是配置整个 Netty 程序，串联各个组件，Netty中 Bootstrap类是客户端程序的启动引导类，ServerBootstrap 是服务端启动引导类。

> 常见的方法有
>
> 1. `public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup)`，该方法用于服务器端，用来设置两个 EventLoop
> 2. `public B group(EventLoopGroup group)`，该方法用于客户端，用来设置一个 EventLoop
> 3. `public B channel(Class<? extends C> channelClass)`，该方法用来设置一个服务器端的通道实现
> 4. `public <T> B option(ChannelOption<T> option, T value)`，用来给 ServerChannel 添加配置
> 5. `public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value)`，用来给接收到的通道添加配置
> 6. `public ServerBootstrap childHandler(ChannelHandler childHandler)`，该方法用来设置业务处理类（自定义的handler）
> 7. `public ChannelFuture bind(int inetPort)`，该方法用于服务器端，用来设置占用的端口号
> 8. `public ChannelFuture connect(String inetHost, int inetPort)`，该方法用于客户端，用来连接服务器端

### 2.2 Future、ChannelFuture

Netty中所有的 IO操作都是异步的，不能立刻得知消息是否被正确处理。但是可以过一会等它执行完成或者直接注册一个监听，具体的实现就是通过Future和 ChannelFutures，他们可以注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件

> 常见的方法有
>
> - `Channel channel()`，返回当前正在进行 `IO` 操作的通道
> - `ChannelFuture sync()`，等待异步操作执行完毕

### 2.3 Channel

Netty 网络通信的组件，能够用于执行网络 I/O 操作。通过 Channel 可获得当前网络连接的通道的状态，通过 Channel 可获得网络连接的配置参数（例如接收缓冲区大小）

Channel 提供异步的网络 I/O 操作(如建立连接，读写，绑定端口)，异步调用意味着任何 I/O 调用都将立即返回，并且不保证在调用结束时所请求的 I/O 操作已完成，调用立即返回一个 

ChannelFuture 实例，通过注册监听器到 ChannelFuture 上，可以 I/O 操作成功、失败或取消时回调通知调用方。支持关联 I/O 操作与对应的处理程序

不同协议、不同的阻塞类型的连接都有不同的 Channel 类型与之对应，常用的 Channel 类型：

> NioSocketChannel，异步的客户端 TCP Socket 连接。
> NioServerSocketChannel，异步的服务器端 TCP Socket 连接。
> NioDatagramChannel，异步的 UDP 连接。
> NioSctpChannel，异步的客户端 Sctp 连接。
> NioSctpServerChannel，异步的 Sctp 服务器端连接，这些通道涵盖了 UDP 和 TCP 网络 IO 以及文件 IO。

### 2.4 ChannelHandler 及其实现类

ChannelHandler是一个接口，处理 I/O 事件或拦截 I/O操作，并将其转发到其 ChannelPipeline（业务处理链）中的下一个处理程序。ChannelHandler本身并没有提供很多方法，因为这个接口有许多方法需要实现，方便使用期间可以继承他的子类。类图如下：

~~~mermaid
 classDiagram
     class ChannelHandler{
        <<interface>>
    }
    ChannelHandler <|-- ChannelOutboundHandler
    ChannelHandler <|-- ChannelInboundHandler
    ChannelHandler <|-- ChannelHandlerAdapter
    class ChannelOutboundHandler{
    <<interface>>
    }
    class ChannelInboundHandler{
    <<interface>>
    }
    class ChannelHandlerAdapter{
    }
    ChannelOutboundHandler <|--ChannelOutboundHandlerAdapter
    ChannelHandlerAdapter <|--ChannelOutboundHandlerAdapter
    ChannelHandlerAdapter <|--ChannelInboundHandlerAdapter
    ChannelInboundHandler <|--ChannelInboundHandlerAdapter
    class ChannelOutboundHandlerAdapter{
    }
    class ChannelInboundHandlerAdapter{
    }
~~~

> 如果不能看到类图请自行下载GitHub+MerMaid浏览器插件，或者下载到本地使用md文本编辑器查看

ChannelInboundHandler用于处理入站IO事件

ChannelOutboundHandler用于处理出战IO操作

ChannelInboundHandlerAdapter用于处理入站IO事件

ChannelOutboundHandlerAdapter用于处理出战IO操作

通常需要自定义一个Handler类去继承ChannelInboundHandlerAdapter，然后通过重写相应的方法来实现业务逻辑。一般需要重写以下方法：

~~~java
//通道就绪事件
public void channelActive(ChannelHandlerContext ctx)throws Exception{
    ctx.fireChannelActive();
}
//通道读取数据事件
public void channelRead(ChannelHandlerContext ctx,Object msg)throws Exception{
    ctx.fireChannelRead(msg);
}
//通道读取完毕事件
public void channelReadComplete(ChannelHandlerContext ctx)throws Exception{
    ctx.fireChannelReadComplete();
}
//通道发生异常事件
public void exceptionCaught(ChannelHandlerContext ctx,Throwable cause)throws Exception{
    ctx.fireChannelexceptionCaught(cause);
}
~~~

### 2.5 Pipeline 和 ChannelPipeline

ChannelPipeline是一个Handler的集合，它负责处理和拦截inbound或者outbound的事件和操作，相当于一个贯穿neyyt的链。（PS：也可以理解为ChannelPipeline` 是保存 `ChannelHandler` 的 `List）

ChannelPipeline实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel中各个的 ChannelHandler如何相互交互。

在 Netty中每个 Channel都有且仅有一个 ChannelPipeline与之对应，它们的组成关系如下：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty10.png)

- 一个Channel包含了一个ChannelPipeline，而ChanelPipeline中有维护者一个由ChannelHandlerContext组成的双向链表，并且每个ChannelHandlerContext中有关联着一个ChannelHandler
- 入站时间和出站事件在一个双向链表中，入站事件会从链表head向后传递一个入站的handler，出站事件会从链表tail向前传递一个出站的handler，两种类型互不干扰。

常用方法：

~~~java
ChannelPipeline addFirst(ChannelHandler... handlers)//把一个业务处理类（handler）添加到链中的第一个位置
 ChannelPipeline addLast(ChannelHandler... handlers)//把一个业务处理类（handler）添加到链中的最后一个位置。
~~~

### 2.6 ChannelHandlerContext

保存Channel相关的所有的上下文信息，同时关联一个ChannelHandler。即 ChannelHandlerContext中包含一个具体的事件处理器 ChannelHandler，同时 ChannelHandlerContext中也绑定了对应的 pipeline和 Channel的信息，方便对 ChannelHandler 进行调用。常用方法：

- `ChannelFuture close()`，关闭通道
- `ChannelOutboundInvoker flush()`，刷新
- `ChannelFuture writeAndFlush(Object msg)`，写数据

### 2.7 ChannelOption

Netty在创建Channel实例后，一般需要设置ChannelOption参数，参数如下：

`ChannelOption.SO_BACKLOG`对应TPC/IP学习listen函数中的backlog参数，用来初始化服务器可连接队列大小，服务端处理客户端连接请求时顺序处理的，所以同一时间只能处理一个客户端连接，多个客户端来的时候，服务端将不能处理客户端连接请求放在队列中等待处理，backlog参数制定了队列的大小

`ChannelOption.SO_KEEPALIVE`一直保持连接活动状态

### 2.8 EventLoopGroup 和其实现类 NioEventLoopGroup

EventLoopGroup 是一组 EventLoop的抽象，Netty为了更好的利用多核 CPU资源，一般会有多个 EventLoop同时工作，每个 EventLoop维护着一个 Selector实例。

EventLoopGroup 提供 next 接口，可以从组里面按照一定规则获取其中一个 EventLoop 来处理任务。在 Netty 服务器端编程中，我们一般都需要提供两个 EventLoopGroup，例如：BossEventLoopGroup 和 WorkerEventLoopGroup。

通常一个服务端口即一个 ServerSocketChannel 对应一个 Selector 和一个 EventLoop 线程。BossEventLoop 负责接收客户端的连接并将 SocketChannel 交给 WorkerEventLoopGroup 来进行 IO 处理，如下图所示：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty11.png)

BossEventLoopGroup通常是一个单线程的EventLoop。EventLoop维护着一个注册了ServerSocketChannel的Selector实例BossEventLoop不断轮询Selector将连接时间分离出来。

通常是OP_ACCEPT事件，然后将接收到的SocketChannel交给WorkerEventLoopGroup

WorkerEventLoopGroup将会由next选择其中一个EventLoop来将这个SocketChannel注册到其维护的Selector并对其后续的IO事件进行处理。

常用方法：

`public NioEventLoopGroup()`，构造方法
`public Future<?> shutdownGracefully()`，断开连接，关闭线程



## 三、自定义RPC

### 3.1 概述

RPC即远程过程调用，是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络实现的技术。常见的RPC框架Dubbo、grpc、SpringCloud等。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/rpc.png)

1. 服务消费方以本地调用方式调用服务
2. client stub 接收到调用后负责将方法、参数等封装成能够进行网络传输的消息体
3. client stub 将消息编码后发送到服务端
4. server stub 收到消息后进行解码
5. server stub根据解码结果调用本地服务
6. 本地服务执行并将结果返回给server stub
7. server stub将返回结果编码并发送到服务消费方
8. client stub接收到消息并进行解码
9. 服务消费方得到结果

### 3.2 设计

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/rpc2.png)

- Client：两个接口+一个包含main方法的测试类
- Client Stub：一个客户端代理类+一个客户端业务处理类
- Server：两个接口+两个实现类
- Server Stub：一个网络处理服务器+一个服务器处理业务类

服务调用方的接口必须跟服务提供方的接口保持一致（包路径可以不同），最终实现在TestNettyRPC中远程调用HelloRPCImpl、HelloNetty中放的方法。

### 3.3 实现

被调用的接口和实现类

```java
public interface HelloNetty{
    String hello();
}
public class HelloNettyImpl implements HelloNetty{
    @Override
    public String hello() {
        return "hello netty";
    }
}
public interface HelloRPC {
    String hello(String name);
}
public class HelloRPCImpl implements HelloRPC{
    @Override
    public String hello(String name) {
        return "hello,"+name;
    }
}
```

服务端业务处理类

```java
public class InvokeHandler extends ChannelInboundHandlerAdapter {

    private String getImplClassName(ClassInfo classInfo) throws Exception {
        String interfacePath = "com.learn.rpcserver";
        int lastDot = classInfo.getClassName().lastIndexOf(".");
        String interfaceName = classInfo.getClassName().substring(lastDot);
        Class superClass = Class.forName(interfacePath + interfaceName);
        Reflections reflections = new Reflections();
        //得到某接口下所有的实现类
        Set<Class> ImplClassSet = reflections.getSubTypesOf(superClass);
        if (ImplClassSet.size()==0){
            System.out.println("未找到实现类");
            return null;
        }else if (ImplClassSet.size()>1){
            System.out.println("找到多个实现类");
            return null;
        }else {
            Class[] classes = ImplClassSet.toArray(new Class[0]);
            return classes[0].getName();
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ClassInfo classInfo = (ClassInfo) msg;
        Object calzz = Class.forName(getImplClassName(classInfo)).newInstance();
        Method method = calzz.getClass().getMethod(classInfo.getMethodName(), classInfo.getTypes());
        Object result = method.invoke(calzz, classInfo.getObjects());
        ctx.writeAndFlush(result);
    }
}
```

服务端

```java
public class NettyRPCServer {

    private int port;

    public NettyRPCServer(int port) {
        this.port = port;
    }

    public void start() throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .localAddress(port)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        //编码器
                        pipeline.addLast("encoder", new ObjectEncoder());
                        //解码器
                        pipeline.addLast("decoder", new ObjectDecoder(Integer.MAX_VALUE, ClassResolvers.cacheDisabled(null)));
                        //服务器端业务处理类
                        pipeline.addLast(new InvokeHandler());
                    }
                });
        ChannelFuture future = bootstrap.bind(port).sync();
        System.out.println("server is readey");
        future.channel().closeFuture().sync();
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }

    public static void main(String[] args) throws InterruptedException {
        new NettyRPCServer(9999).start();
    }
}
```

数据传输类

```java
public class ClassInfo implements Serializable {
    private static final long serialVersionUID =1L;
    private String className;//类名
    private String methodName;//方法名
    private Class<?>[] types;//参数类型
    private Object[] objects;//参数列表
    //get and set
}
```

客户端代理类

```java
public class NettyRPCProxy {

    public static Object create(Class target){
        return Proxy.newProxyInstance(target.getClassLoader(), new Class[]{target}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                //封装classinfo
                ClassInfo classInfo = new ClassInfo();
                classInfo.setClassName(target.getName());
                classInfo.setMethodName(method.getName());
                classInfo.setObjects(args);
                classInfo.setTypes(method.getParameterTypes());
                //开始用netty发送数据
                EventLoopGroup group = new NioEventLoopGroup();
                ResultHandler resultHandler = new ResultHandler();
                Bootstrap bootstrap =new Bootstrap();
                bootstrap.group(group)
                        .channel(NioSocketChannel.class)
                        .handler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            protected void initChannel(SocketChannel ch) throws Exception {
                                ChannelPipeline pipeline = ch.pipeline();
                                //编码器
                                pipeline.addLast("encoder",new ObjectEncoder());
                                //解码器
                                pipeline.addLast("decoder",new ObjectDecoder(Integer.MAX_VALUE, ClassResolvers.cacheDisabled(null)));
                                pipeline.addLast("handler",resultHandler);
                            }
                        });
                ChannelFuture future = bootstrap.connect("127.0.0.1", 9999).sync();
                future.channel().writeAndFlush(classInfo).sync();
                future.channel().closeFuture().sync();
                group.shutdownGracefully();
                return resultHandler.getResponse();
            }
        });
    }
}
```

客户端服务处理

```java
public class ResultHandler extends ChannelInboundHandlerAdapter {
    private Object response;

    public Object getResponse() {
        return response;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        response = msg;
        ctx.close();
    }
}
```

客户端测试

```java
public class TestNettyRpc {
    public static void main(String[] args) {
        //第一次远程调用
        HelloNetty helloNetty = (HelloNetty) NettyRPCProxy.create(HelloNetty.class);
        System.out.println(helloNetty.hello());
        //第二次远程调用
        HelloRPC helloRPC = (HelloRPC) NettyRPCProxy.create(HelloRPC.class);
        System.out.println(helloRPC.hello("hahahahaha"));
    }
}
```

