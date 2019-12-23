# java IO之路

## linux io

linux的内核将所有的外部设备都看作一个文件，对一个文件的读写都对应着内核提供的命令，返回一个fd，对socket的读写也会返回相应的表示符，socketfd,  

unix网络对Io模型进行了分类，大致分为5类  
一： 阻塞IO ，最常用的io模型就是阻塞IO模型，所有的文件读写操作都是阻塞的。在socket中，在发送数据，到达，存储才会返回成功或者失败，在这一段时间会一直等待，不会进行其他操作，所以他是阻塞的  
二： 非阻塞IO模型： 加入了缓冲区，如果取数据的时候发现没有数据或者失败会直接返回  
三： IO复用模型： linux提供select/poll，进程将fd传递给select或者poll系统调用，阻塞在select上，select/poll检测多个fd是否处于就绪状态，然后扫描，epoll事件驱动  
四：信号驱动IO模型，两端通过信号来处理，当发送方准备就绪的时候会发出信号，接收方会去读取数据  
五: 异步io模型 ： 告知内核启动某个操作，内核启动了然后通知已完成。  

## NIO

### Bio
网络编程中主要是Client/server 模型，也就是两根线程进行通信，server暴露ip，client连接，通过三次握手建立连接，如果建立成功，双方可以通过socket通信  
通讯结构大致假如多个客户端去链接server，服务端就会有多少根线程与他们一一对应链接，这样如果客户端过多，服务端启动线程过多，每个线程都会占用内存，这样会导致内存票高，系统的性能急剧下降，最终会导致宕机等  

### 伪异步io 
采用线程池来代替多线程，但是没从根本上解决阻塞的问题。当线程池处理不过来的时候，会导致数据的处理缓慢  

### NIO
nio是java 1.4推出的new IO ，他在java代码中提供了高速的面向块的io。

#### 缓冲区 buffer
buffer是一个对象，他包含了要写入和读出的数据，这是nio和以前的io的一个重要区别，所有数据都是直接写入或者读入缓冲区。  
缓冲区实际上是一个数组，通常是一个字节数组。每个java基本类型都对应了一种数组。  

#### 通道channel
channel是一个通道，通过他读和写数据。通道和流的不同之处在于，流是单向的，通道时双向的。通道可以用来读或者写  
因为channel是全双工的，他可以更好的反映底层操作系统的api，在unix系统中，底层的通道都是全双工的 

#### 多路复用 Selector  
selector 不断轮询注册在上面的channel,如果某个channel有新的tcp接入或者读写，这个channel都会处于就绪状态。会被selector轮询出来，通过selectKey选择出来，进行IO操作  

#### coding 

server端
```java
public class MyNioTest {

    static ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) Executors.newFixedThreadPool(5);

    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        //new 一个channel并设置通道，地址
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(new InetSocketAddress("127.0.0.1",8080));
        serverSocketChannel.configureBlocking(false);
        //注册到channel
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true){
            selector.select(1000);
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                //如果不移除，原来的key 会报空指针
                iterator.remove();
                if(key.isValid()){
                    //新接入
                    if(key.isAcceptable()){
                        ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                        SocketChannel accept = channel.accept();
                        accept.configureBlocking(false);
                        //必须转换，serverSocketChannetl只能accect，不能写，读
                        accept.register(selector,SelectionKey.OP_READ);
                    }
                    if(key.isReadable()){
                        SocketChannel channel = (SocketChannel) key.channel();
                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);//申请内存  1KB
                        int read = channel.read(byteBuffer);
                        if(read>0){
                            byteBuffer.flip();
                            byte[] bytes = new byte[byteBuffer.remaining()];
                            byteBuffer.get(bytes);
                            String body = new String(bytes,"utf-8");
                            handleTask(body);
                            channel.write(ByteBuffer.wrap("ok".getBytes()));
                        }else if(read<0){
                            key.cancel();
                            serverSocketChannel.close();
                        }
                    }
                }
            }
        }
    }

    public static void handleTask(String task){
        threadPoolExecutor.execute(()->{
            System.out.println(task);
        });
    }
}
```

Client 
```java
public class NioClientTest {

    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        boolean connect = socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
        if(connect){
            socketChannel.register(selector,SelectionKey.OP_READ);
        }else{
            socketChannel.register(selector, SelectionKey.OP_CONNECT);
        }
        while (true){
            selector.select(1000);
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                iterator.remove();
                if(key.isValid()){
                    if(key.isConnectable()){
                        SocketChannel channel = (SocketChannel)key.channel();
                        if(channel.finishConnect()){
                            System.out.println("connect success");
                            socketChannel.register(selector,SelectionKey.OP_READ|SelectionKey.OP_WRITE);
                            channel.write(ByteBuffer.wrap("success connect ".getBytes()));
                        }
                    }
                    if(key.isReadable()){
                        SocketChannel channel = (SocketChannel) key.channel();
                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);//申请内存  1KB
                        int read = channel.read(byteBuffer);
                        if(read>0){
                            byteBuffer.flip();
                            byte[] bytes = new byte[byteBuffer.remaining()];
                            byteBuffer.get(bytes);
                            String body = new String(bytes,"utf-8");
                            System.out.println(body);
                            channel.write(ByteBuffer.wrap("ok".getBytes()));
                        }
                    }
                }
            }
        }
    }
}
```

## AIO编程

java 在jdk1.7中引入了AIO这个概念，也就是NIO2，并提供了异步文件和socket的实现  
AIO是真正的异步非阻塞io，他对应unix中的事件驱动，不需要selector对注册通道沦胥实现异步的读写，从而简化了NIO变成模型  

### Coding

#### server端

server端 
```java
public class MyNio2ServerTest {

    static CountDownLatch countDownLatch = new CountDownLatch(1);

    public static void main(String[] args) throws IOException, InterruptedException, ExecutionException {
        int port = 8082;
        AsynchronousServerSocketChannel asynchronousServerSocketChannel = AsynchronousServerSocketChannel.open();
        asynchronousServerSocketChannel.bind(new InetSocketAddress("127.0.0.1",port));
        System.out.println("the channel is open");
        while (true){
            doAcceptUseFuture(asynchronousServerSocketChannel);
        }
//        countDownLatch.await();
    }

    private static void doAcceptUseFuture(AsynchronousServerSocketChannel asynchronousServerSocketChannel) throws ExecutionException, InterruptedException, UnsupportedEncodingException {
        Future<AsynchronousSocketChannel> accept = asynchronousServerSocketChannel.accept();
        AsynchronousSocketChannel asynchronousSocketChannel = accept.get();
        ByteBuffer byteBuffer = ByteBuffer.allocate(2);
        byteBuffer.clear();
        while (true){
            Integer num = asynchronousSocketChannel.read(byteBuffer).get();
            System.out.println(num);
            if(num == -1){
                break;
            }
            byteBuffer.flip();
            System.out.println(new String(byteBuffer.array(), "utf-8"));
            byteBuffer.clear();
        }


//            asynchronousServerSocketChannel.accept(new MyNio2ServerTest(), new CompletionHandler<AsynchronousSocketChannel, MyNio2ServerTest>() {
//                    @Override
//                    public void completed(AsynchronousSocketChannel result, MyNio2ServerTest attachment) {
//                        System.out.println("completed");
//                        asynchronousServerSocketChannel.accept(attachment,this);
//                    }
//
//                    @Override
//                    public void failed(Throwable exc, MyNio2ServerTest attachment) {
//                        System.out.println("failed");
//                        countDownLatch.countDown();
//                    }
//            });
    }
}
```

## 4种IO的对比

###异步非阻塞IO

很多人喜欢将NIO成为异步非阻塞IO，严格的说他只能成为非阻塞IO，不能称为异步非阻塞IO。  
jdk1.5 update10版本以前，NIO使用的底层是select/poll.后来使用epoll，但是还不是异步非阻塞IO。
只是在以前的基础上做出了优化  
JDK中NIO2才是真正的异步非阻塞IO。

### 对比

| | 同步阻塞IO（BIO） | 伪异步IO | 非阻塞IO（NIO） |异步IO（AIO）
| ------ | ------ | ------ |------|------|
| 客户端个数 | 1：1 | M:N |M：1|M：0|
| 是否阻塞 | 是 | 是 |否|否|
| 是否同步 | 是 | 是 |是|否|
| 是否简单 | 是 | 是 |否|否|
| 调试难度 | 易 | 易 |难|难|
| 可靠性 | 非常差 | 差 |高|高|
| 吞吐量 | 低 | 低 |高|高|




