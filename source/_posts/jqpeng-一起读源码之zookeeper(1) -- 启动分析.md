---
title: 一起读源码之zookeeper(1) -- 启动分析
tags: ["zookeeper","java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-12-01 14:43
---
文章作者:jqpeng
原文链接: [一起读源码之zookeeper(1) -- 启动分析](https://www.cnblogs.com/xiaoqi/p/7942234.html)

从本文开始，不定期分析一个开源项目源代码，起篇从大名鼎鼎的zookeeper开始。  
 为什么是zk，因为用到zk的场景实在太多了，大部分耳熟能详的分布式系统都有zookeeper的影子，比如hbase，storm，dubbo，kafka等等，另外前面提到的[RPC框架原理与实现](http://www.cnblogs.com/xiaoqi/p/java-rpc.html)也用到了zookeeper。




目录

- 1 环境准备
    - 1.1 导入代码
    - 1.2 设置配置文件
    - 1.3 调试配置
- 2 启动分析
    - 2.1 QuorumPeerMain
    - 2.2 ZooKeeperServerMain
    - 2.3 ServerCnxnFactory
    - 2.4 ZooKeeperServer
    - 2.5 服务启动
        - 2.5.1 配置cnxnFactory
        - 2.5.2 启动cnxnFactory
            - socket处理线程
            - socket网络请求处理
            - 读取连接请求
            - 创建session
        - 2.5.3 zk服务器启动
            - SessionTracker
        - 2.5.4  ZooKeeperServer请求处理器链介绍
            - RequestProcessor
            - PrepRequestProcessor
            - SyncRequestProcessor
            - FinalRequestProcessor





## 1 环境准备

首先，下载zk的新版本，最新的稳定版是3.4.10，由于已下载3.4.9.先直接使用。

### 1.1 导入代码

IDEA直接打开zk目录：  
![enter description here](https://ooo.0o0.ooo/2017/04/27/590166a796fb3.jpg "1493264040459")

项目设置为jdk1.7  
 然后，将src/java下面的main和generated设置为源码目录，同时将lib目录添加为liabary。

### 1.2 设置配置文件

在conf目录，新建zoo.cfg，拷贝sample.cfg即可

![enter description here](https://ooo.0o0.ooo/2017/04/27/5901674b5600d.jpg "1493264204261")

### 1.3 调试配置

查看bin/zkServer


    set ZOOMAIN=org.apache.zookeeper.server.quorum.QuorumPeerMain
    ....
    endlocal


调用的是org.apache.zookeeper.server.quorum.QuorumPeerMain，因此QuorumPeerMain，配置调试程序，arguments设置conf/zoo.cfg

![enter description here](https://ooo.0o0.ooo/2017/04/27/5901838aed112.jpg "1493271435981")

这样，就可以愉快的Debug代码了-😃

## 2 启动分析

### 2.1 QuorumPeerMain

QuorumPeerMain的main里，调用initializeAndRun


        protected void initializeAndRun(String[] args)
            throws ConfigException, IOException
        {
            QuorumPeerConfig config = new QuorumPeerConfig();
            if (args.length == 1) {
                config.parse(args[0]);
            }
    
            // Start and schedule the the purge task 清理任务
            DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                    .getDataDir(), config.getDataLogDir(), config
                    .getSnapRetainCount(), config.getPurgeInterval());
            purgeMgr.start();
    
            // 集群模式
            if (args.length == 1 && config.servers.size() > 0) {
                runFromConfig(config);
            } else {
                LOG.warn("Either no config or no quorum defined in config, running "
                        + " in standalone mode");
                // there is only server in the quorum -- run as standalone
                // 单机模式
                ZooKeeperServerMain.main(args);
            }
        }


主要执行了：

- 加载解析配置文件到QuorumPeerConfig
- 执行清理任务
- 判断是集群模式还是单机模式，我们的配置文件未配置server，所以是单机模式，执行 ZooKeeperServerMain.main



> 本文重点分析单机模式下的zk，集群模式暂时不解读


### 2.2 ZooKeeperServerMain

ZooKeeperServerMain.main调用initializeAndRun


     protected void initializeAndRun(String[] args)
            throws ConfigException, IOException
        {
            try {
                ManagedUtil.registerLog4jMBeans();
            } catch (JMException e) {
                LOG.warn("Unable to register log4j JMX control", e);
            }
    
            ServerConfig config = new ServerConfig();
            if (args.length == 1) {
                config.parse(args[0]);
            } else {
                config.parse(args);
            }
    
            runFromConfig(config);
        }```
    
    读取配置，然后runFromConfig：
    
    ``` java
     public void runFromConfig(ServerConfig config) throws IOException {
            LOG.info("Starting server");
            FileTxnSnapLog txnLog = null;
            try {
                // Note that this thread isn't going to be doing anything else,
                // so rather than spawning another thread, we will just call
                // run() in this thread.
                // create a file logger url from the command line args
                final ZooKeeperServer zkServer = new ZooKeeperServer();
                // Registers shutdown handler which will be used to know the
                // server error or shutdown state changes.
                final CountDownLatch shutdownLatch = new CountDownLatch(1);
                zkServer.registerServerShutdownHandler(
                        new ZooKeeperServerShutdownHandler(shutdownLatch));
    
                // 快照
                txnLog = new FileTxnSnapLog(new File(config.dataLogDir), new File(
                        config.dataDir));
                zkServer.setTxnLogFactory(txnLog);
                zkServer.setTickTime(config.tickTime);
                zkServer.setMinSessionTimeout(config.minSessionTimeout);
                zkServer.setMaxSessionTimeout(config.maxSessionTimeout);
                // socket工厂
                cnxnFactory = ServerCnxnFactory.createFactory();
                cnxnFactory.configure(config.getClientPortAddress(),
                        config.getMaxClientCnxns());
                cnxnFactory.startup(zkServer);
    
                // Watch status of ZooKeeper server. It will do a graceful shutdown
                // if the server is not running or hits an internal error.
                shutdownLatch.await();
                shutdown();
    
                cnxnFactory.join();
                if (zkServer.canShutdown()) {
                    zkServer.shutdown();
                }
            } catch (InterruptedException e) {
                // warn, but generally this is ok
                LOG.warn("Server interrupted", e);
            } finally {
                if (txnLog != null) {
                    txnLog.close();
                }
            }
        }


几件事情：

- 创建zkServer，对ZooKeeperServer设置一些配置参数，如tickTime、minSessionTimeout、maxSessionTimeout
- 创建CountDownLatch，注释里写了，用来watch zk的状态，当zk关闭或者出现内部错误的时候**优雅**的关闭服务
- 根据配置参数dataLogDir和dataDir创建FileTxnSnapLog，用来存储zk数据和日志快照
- 创建cnxnFactory，zk的 socket工厂，负责处理网络请求，zk里有netty和NIO两种实现
- cnxnFactory.startup(zkServer)，启动zk服务器


### 2.3 ServerCnxnFactory

cnxnFactory负责zk的网络请求，createFactory中，从系统配置中读取ZOOKEEPER\_SERVER\_CNXN\_FACTORY，默认是没有这个配置的，因此默认是使用NIOServerCnxnFactory，基于java的NIO实现，


        static public ServerCnxnFactory createFactory() throws IOException {
            String serverCnxnFactoryName =
                System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
            if (serverCnxnFactoryName == null) {
                serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
            }
            try {
                return (ServerCnxnFactory) Class.forName(serverCnxnFactoryName)
                                                    .newInstance();
            } catch (Exception e) {
                IOException ioe = new IOException("Couldn't instantiate "
                        + serverCnxnFactoryName);
                ioe.initCause(e);
                throw ioe;
            }
        }


当然，我们可以很容易发现：  
![enter description here](https://ooo.0o0.ooo/2017/04/27/59018f04f17b9.jpg "1493274374075")

ServerCnxnFactory还有个NettyServerCnxnFactory实现，基于Netty实现NIO。ServerCnxnFactory里具体负责什么，后面再来看。

### 2.4 ZooKeeperServer

现在，主角登场，我们来看ZooKeeperServer内部有什么玄妙。  
![enter description here](https://ooo.0o0.ooo/2017/04/27/59018cc60446f.jpg "1493273799033")

ZooKeeperServer是单机模式使用的类，在集群模式下使用的是它的子类。  
 我们先来看ZooKeeperServer包含哪些内容：


        public static final int DEFAULT_TICK_TIME = 3000;
        protected int tickTime = DEFAULT_TICK_TIME;
        /** value of -1 indicates unset, use default */
        protected int minSessionTimeout = -1;
        /** value of -1 indicates unset, use default */
        protected int maxSessionTimeout = -1;
        protected SessionTracker sessionTracker; //创建和管理session
        private FileTxnSnapLog txnLogFactory = null; //文件快照
        private ZKDatabase zkDb; // ZooKeeper树形数据的模型
        private final AtomicLong hzxid = new AtomicLong(0); //原子增长Long，用于分配事务编号
        public final static Exception ok = new Exception("No prob");
        protected RequestProcessor firstProcessor; // ZooKeeperServer请求处理器链中的第一个处理器
        protected volatile State state = State.INITIAL;
    
        protected enum State {
            INITIAL, RUNNING, SHUTDOWN, ERROR;
        }
    
        /**
         * This is the secret that we use to generate passwords, for the moment it
         * is more of a sanity check.
         */
        static final private long superSecret = 0XB3415C00L;
    
        private final AtomicInteger requestsInProcess = new AtomicInteger(0);
        final List<ChangeRecord> outstandingChanges = new ArrayList<ChangeRecord>();
        // this data structure must be accessed under the outstandingChanges lock
        final HashMap<String, ChangeRecord> outstandingChangesForPath =
            new HashMap<String, ChangeRecord>();
        
        private ServerCnxnFactory serverCnxnFactory; //ServerSocket工厂，接受客户端的socket连接
    
        private final ServerStats serverStats; //server的运行状态统计
        private final ZooKeeperServerListener listener; // ZK运行状态监听
        private ZooKeeperServerShutdownHandler zkShutdownHandler;


### 2.5 服务启动

前面有点跑偏，继续回归启动过程:


                cnxnFactory = ServerCnxnFactory.createFactory();
                cnxnFactory.configure(config.getClientPortAddress(),
                        config.getMaxClientCnxns());
                cnxnFactory.startup(zkServer);


#### 2.5.1 配置cnxnFactory

进入configure：


        @Override
        public void configure(InetSocketAddress addr, int maxcc) throws IOException {
            configureSaslLogin();
    
            // ZK网络请求主线程
            thread = new ZooKeeperThread(this, "NIOServerCxn.Factory:" + addr);
            thread.setDaemon(true);
    
            maxClientCnxns = maxcc;
            this.ss = ServerSocketChannel.open();
            ss.socket().setReuseAddress(true);
            LOG.info("binding to port " + addr);
            ss.socket().bind(addr);
            ss.configureBlocking(false);
            ss.register(selector, SelectionKey.OP_ACCEPT);
        }


几件事情：

- configureSaslLogin，具体不细看，应该是处理鉴权
- 初始化ZooKeeperThread，这个ZooKeeperThread的作用是负责处理未处理异常：



    public class ZooKeeperThread extends Thread {
    
        private static final Logger LOG = LoggerFactory
                .getLogger(ZooKeeperThread.class);
    
        private UncaughtExceptionHandler uncaughtExceptionalHandler = new UncaughtExceptionHandler() {
    
            @Override
            public void uncaughtException(Thread t, Throwable e) {
                handleException(t.getName(), e);
            }
        };
    
        public ZooKeeperThread(Runnable thread, String threadName) {
            super(thread, threadName);
            setUncaughtExceptionHandler(uncaughtExceptionalHandler);
        }
    
        protected void handleException(String thName, Throwable e) {
            LOG.warn("Exception occured from thread {}", thName, e);
        }
    }


- 启动ServerSocketChannel，并绑定配置的addr，并且注册selector（可以搜索NIO了解细节）


#### 2.5.2 启动cnxnFactory

继续分析，进入cnxnFactory.startup(zkServer)


        @Override
        public void startup(ZooKeeperServer zks) throws IOException,
                InterruptedException {
            start();
            setZooKeeperServer(zks);
            zks.startdata();
            zks.startup();
        }


首先，start，判断线程状态，如果未启动则启动线程，注意只会启动一次。


        @Override
        public void start() {
            // ensure thread is started once and only once
            if (thread.getState() == Thread.State.NEW) {
                thread.start();
            }
        }


##### socket处理线程

启动后，就会执行cnxnFactory.run


        public void run() {
            while (!ss.socket().isClosed()) {
                try {
                    selector.select(1000);
                    Set<SelectionKey> selected;
                    synchronized (this) {
                        selected = selector.selectedKeys();
                    }
                    ArrayList<SelectionKey> selectedList = new ArrayList<SelectionKey>(
                            selected);
                    Collections.shuffle(selectedList);
                    for (SelectionKey k : selectedList) {
                        if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                            SocketChannel sc = ((ServerSocketChannel) k
                                    .channel()).accept();
                            InetAddress ia = sc.socket().getInetAddress();
                            int cnxncount = getClientCnxnCount(ia);
                            if (maxClientCnxns > 0 && cnxncount >= maxClientCnxns){
                                LOG.warn("Too many connections from " + ia
                                         + " - max is " + maxClientCnxns );
                                sc.close();
                            } else {
                                LOG.info("Accepted socket connection from "
                                         + sc.socket().getRemoteSocketAddress());
                                sc.configureBlocking(false);
                                SelectionKey sk = sc.register(selector,
                                        SelectionKey.OP_READ);
                                NIOServerCnxn cnxn = createConnection(sc, sk);
                                sk.attach(cnxn);
                                addCnxn(cnxn);
                            }
                        } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                            NIOServerCnxn c = (NIOServerCnxn) k.attachment();
                            c.doIO(k);
                        } else {
                            if (LOG.isDebugEnabled()) {
                                LOG.debug("Unexpected ops in select "
                                          + k.readyOps());
                            }
                        }
                    }
                    selected.clear();
                } catch (RuntimeException e) {
                    LOG.warn("Ignoring unexpected runtime exception", e);
                } catch (Exception e) {
                    LOG.warn("Ignoring exception", e);
                }
            }
            closeAll();
            LOG.info("NIOServerCnxn factory exited run method");
        }


这里相当于一个独立线程来处理网络连接，通过selector.select(1000)来获取网络请求，一旦有连接就绪，则开始处理：

- 首先打乱 Collections.shuffle(selectedList);
- for循环处理
    - 如果SelectionKey.OP\_ACCEPT，代表一个新连接请求，创建SocketChannel，创建NIOServerCnxn，然后addCnxn
    - 如果可读写，则 NIOServerCnxn.doIO(k)，执行IO操作


##### socket网络请求处理

这里简单分析下doIO,摘录部分代码：


    void doIO(SelectionKey k) throws InterruptedException {
            try {
                if (isSocketOpen() == false) {
                    LOG.warn("trying to do i/o on a null socket for session:0x"
                             + Long.toHexString(sessionId));
    
                    return;
                }
                if (k.isReadable()) {
                    // 读取4个字节
                    int rc = sock.read(incomingBuffer);
                    if (rc < 0) {
                        throw new EndOfStreamException(
                                "Unable to read additional data from client sessionid 0x"
                                + Long.toHexString(sessionId)
                                + ", likely client has closed socket");
                    }
                    // 读满了
                    if (incomingBuffer.remaining() == 0) {
                        boolean isPayload;
                        if (incomingBuffer == lenBuffer) { // start of next request
                            incomingBuffer.flip(); // 复位
                            isPayload = readLength(k); // 读取载荷长度
                            incomingBuffer.clear();
                        } else {
                            // continuation
                            isPayload = true;
                        }
                        if (isPayload) { // not the case for 4letterword
                            readPayload();
                        }
                        else {
                            // four letter words take care
                            // need not do anything else
                            return;
                        }
                    }
                }


读取4个字节，获取到数据长度，然后读取载荷，也就是请求


        private void readPayload() throws IOException, InterruptedException {
            if (incomingBuffer.remaining() != 0) { // have we read length bytes?
                int rc = sock.read(incomingBuffer); // sock is non-blocking, so ok
                if (rc < 0) {
                    throw new EndOfStreamException(
                            "Unable to read additional data from client sessionid 0x"
                            + Long.toHexString(sessionId)
                            + ", likely client has closed socket");
                }
            }
    
            if (incomingBuffer.remaining() == 0) { // have we read length bytes?
                packetReceived();
                incomingBuffer.flip(); // 复位
                if (!initialized) {
                    readConnectRequest(); // 读取连接请求
                } else {
                    readRequest();
                }
                lenBuffer.clear();
                incomingBuffer = lenBuffer;
            }
        }


先是读取数据，然后再读取请求，这里关注readConnectRequest

##### 读取连接请求


        private void readConnectRequest() throws IOException, InterruptedException {
            if (zkServer == null) {
                throw new IOException("ZooKeeperServer not running");
            }
            zkServer.processConnectRequest(this, incomingBuffer);
            initialized = true;
        }


继续，下面是处理连接请求：


         public void processConnectRequest(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
            BinaryInputArchive bia = BinaryInputArchive.getArchive(new ByteBufferInputStream(incomingBuffer));
            ConnectRequest connReq = new ConnectRequest();
            connReq.deserialize(bia, "connect"); // 反序列化请求
            ....
            // 客户端设置的超时时间
            int sessionTimeout = connReq.getTimeOut();
            byte passwd[] = connReq.getPasswd();
            int minSessionTimeout = getMinSessionTimeout();
            if (sessionTimeout < minSessionTimeout) {
                sessionTimeout = minSessionTimeout;
            }
            // 服务端设置的最大超时时间
            int maxSessionTimeout = getMaxSessionTimeout();
            if (sessionTimeout > maxSessionTimeout) {
                sessionTimeout = maxSessionTimeout;
            }
            cnxn.setSessionTimeout(sessionTimeout);
            // We don't want to receive any packets until we are sure that the
            // session is setup
            cnxn.disableRecv();
            // 请求是否带上sessionid
            long sessionId = connReq.getSessionId();
            if (sessionId != 0) {
                // 请求带了sessionid
                long clientSessionId = connReq.getSessionId();
                LOG.info("Client attempting to renew session 0x"
                        + Long.toHexString(clientSessionId)
                        + " at " + cnxn.getRemoteSocketAddress());
                // 关闭请求
                serverCnxnFactory.closeSession(sessionId);
                cnxn.setSessionId(sessionId);
                // 重新打开请求
                reopenSession(cnxn, sessionId, passwd, sessionTimeout);
            } else {
                LOG.info("Client attempting to establish new session at "
                        + cnxn.getRemoteSocketAddress());
                // 创建新sesssion
                createSession(cnxn, passwd, sessionTimeout);
            }
        }


以上完成：

- 将读取出来的incomingBuffer反序列化为ConnectRequest对象
- 然后设置超时时间，ServerCnxn接收到该申请后，根据客户端传递过来的sessionTimeout时间以及ZooKeeperServer本身的minSessionTimeout、maxSessionTimeout参数，确定最终的sessionTimeout时间
- 判断客户端的请求是否已经含有sessionId
    - 如果含有，则执行sessionId的是否过期、密码是否正确等检查
    - 如果没有sessionId，则创建一个session


##### 创建session

所以，我们需要再看一下如何创建session：


        long createSession(ServerCnxn cnxn, byte passwd[], int timeout) {
            long sessionId = sessionTracker.createSession(timeout);
            Random r = new Random(sessionId ^ superSecret);
            r.nextBytes(passwd);
            ByteBuffer to = ByteBuffer.allocate(4);
            to.putInt(timeout);
            cnxn.setSessionId(sessionId);
            submitRequest(cnxn, sessionId, OpCode.createSession, 0, to, null);
            return sessionId;
        }


- 使用sessionTracker生成一个sessionId
- submitRequest构建一个Request请求，请求的类型为OpCode.createSession



        private void submitRequest(ServerCnxn cnxn, long sessionId, int type,
                int xid, ByteBuffer bb, List<Id> authInfo) {
            Request si = new Request(cnxn, sessionId, xid, type, bb, authInfo);
            submitRequest(si);
        }
        
        public void submitRequest(Request si) {
            if (firstProcessor == null) {
                synchronized (this) {
                    try {
                        // Since all requests are passed to the request
                        // processor it should wait for setting up the request
                        // processor chain. The state will be updated to RUNNING
                        // after the setup.
                        while (state == State.INITIAL) {
                            wait(1000);
                        }
                    } catch (InterruptedException e) {
                        LOG.warn("Unexpected interruption", e);
                    }
                    if (firstProcessor == null || state != State.RUNNING) {
                        throw new RuntimeException("Not started");
                    }
                }
            }
            try {
                touch(si.cnxn);
                boolean validpacket = Request.isValid(si.type);
                if (validpacket) {
                    firstProcessor.processRequest(si);
                    if (si.cnxn != null) {
                        incInProcess();
                    }
                } else {
                    LOG.warn("Received packet at server of unknown type " + si.type);
                    new UnimplementedRequestProcessor().processRequest(si);
                }
            } catch (MissingSessionException e) {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Dropping request: " + e.getMessage());
                }
            } catch (RequestProcessorException e) {
                LOG.error("Unable to process request:" + e.getMessage(), e);
            }
        }


上面的代码：

- 创建一个Request
- 等待firstProcessor创建完成，然后调用firstProcessor.processRequest



> firstProcessor是什么东东，下面再揭晓


#### 2.5.3 zk服务器启动

再次回到startup，  setZooKeeperServer(zks)，代码很简单


     final public void setZooKeeperServer(ZooKeeperServer zk) {
            this.zkServer = zk;
            if (zk != null) {
                zk.setServerCnxnFactory(this);
            }
        }


然后是zk服务器的startdata:


        public void startdata() 
        throws IOException, InterruptedException {
            //check to see if zkDb is not null
            if (zkDb == null) {
                zkDb = new ZKDatabase(this.txnLogFactory);
            }  
            if (!zkDb.isInitialized()) {
                loadData();
            }
        }


初始化ZKDatabase，从txnLogFactory里读取快照数据。

最后是zk服务器的startup：


        public synchronized void startup() {
            if (sessionTracker == null) {
                createSessionTracker();
            }
            startSessionTracker();
            setupRequestProcessors();
    
            registerJMX();
    
            setState(State.RUNNING);
            notifyAll();
        }


几件事情：

- createSessionTracker创建sessionTracker
- startSessionTracker启动SessionTracker
- setupRequestProcessors 创建请求处理器链
- registerJMX 注册JMX
- setState(State.RUNNING) 设置状态为运行中


##### SessionTracker

看SessionTracker的注释：


> This is the basic interface that ZooKeeperServer uses to track sessions.  
>  负责追踪Session的


在zk里的实现是SessionTrackerImpl：


        protected void createSessionTracker() {
            sessionTracker = new SessionTrackerImpl(this, zkDb.getSessionWithTimeOuts(),
                    tickTime, 1, getZooKeeperServerListener());
        }
        
        protected void startSessionTracker() {
            ((SessionTrackerImpl)sessionTracker).start();
        }


SessionTrackerImpl后面再详细分析。

#### 2.5.4  ZooKeeperServer请求处理器链介绍

这里是zk的核心部分之一，zk接收到的请求最终在这里进行处理。


     protected void setupRequestProcessors() {
            RequestProcessor finalProcessor = new FinalRequestProcessor(this);
            RequestProcessor syncProcessor = new SyncRequestProcessor(this,
                    finalProcessor);
            ((SyncRequestProcessor)syncProcessor).start();
            firstProcessor = new PrepRequestProcessor(this, syncProcessor);
            ((PrepRequestProcessor)firstProcessor).start();
        }


请求处理链介绍

- 首先是PrepRequestProcessor
- 然后是SyncRequestProcessor
- 最后是finalProcessor


下面依次解读：

##### RequestProcessor


> RequestProcessors are chained together to process transactions.  
>  RequestProcessors都是链在一起的事务处理链



    public interface RequestProcessor {
        @SuppressWarnings("serial")
        public static class RequestProcessorException extends Exception {
            public RequestProcessorException(String msg, Throwable t) {
                super(msg, t);
            }
        }
    
        void processRequest(Request request) throws RequestProcessorException;
    
        void shutdown();
    }


包含下面这些实现：  
![enter description here](https://ooo.0o0.ooo/2017/04/27/5901abdf8ffaa.jpg "1493281760211")  
 我们重点来看下面几个：

##### PrepRequestProcessor

为什么成为请求处理链，看下PrepRequestProcessor代码就知道了：


        RequestProcessor nextProcessor;
    
        ZooKeeperServer zks;
    
        public PrepRequestProcessor(ZooKeeperServer zks,
                RequestProcessor nextProcessor) {
            super("ProcessThread(sid:" + zks.getServerId() + " cport:"
                    + zks.getClientPort() + "):", zks.getZooKeeperServerListener());
            this.nextProcessor = nextProcessor;
            this.zks = zks;
        }protected void pRequest(Request request) throws RequestProcessorException {
            ……
            nextProcessor.processRequest(request);
        }


构造函数里包含nextProcessor，在pRequest完成后，执行nextProcessor.processRequest，相当于链式执行。

接着分析，再来看类的定义：


    public class PrepRequestProcessor extends ZooKeeperCriticalThread implements
                RequestProcessor {
    
            LinkedBlockingQueue<Request> submittedRequests = new LinkedBlockingQueue<Request>();
    
            RequestProcessor nextProcessor;	
    }
    


几个要点

- 继承自ZooKeeperCriticalThread，是一个Thread
- 重要属性submittedRequests 是一个LinkedBlockingQueue，LinkedBlockingQueue实现是线程安全的，实现了先进先出特性，是作为生产者消费者的首选。


PrepRequestProcessor作为处理链的源头，对外提供processRequest方法收集请求，由于是单线程，所以需要将请求放入submittedRequests队列。


        public void processRequest(Request request) {
            // request.addRQRec(">prep="+zks.outstandingChanges.size());
            submittedRequests.add(request);
        }
    


放入队列后，PrepRequestProcessor本身就是一个Thread，所以start后执行run，在run方法中又会将用户提交的请求取出来进行处理：


        public void run() {
                while (true) {
                    // 取出一个请求
                    Request request = submittedRequests.take();
                    if (Request.requestOfDeath == request) {
                        break;
                    }
                    // 处理请求
                    pRequest(request);
                }
            }


再来看pRequest：  
![enter description here](https://ooo.0o0.ooo/2017/04/27/5901af5b683c8.jpg "1493282652397")

根据request的type，构造对应的请求，对于增删改等影响数据状态的操作都被认为是事务（txn:transaction) ，需要创建出事务请求头(hdr)，调用pRequest2Txn，其他操作则不属于事务操作，需要验证下sessionId是否合法。


     //create/close session don't require request record
                case OpCode.createSession:
                case OpCode.closeSession:
                    pRequest2Txn(request.type, zks.getNextZxid(), request, null, true);
                    break;
     
                //All the rest don't need to create a Txn - just verify session
                case OpCode.sync:
                case OpCode.exists:
                case OpCode.getData:
                case OpCode.getACL:
                case OpCode.getChildren:
                case OpCode.getChildren2:
                case OpCode.ping:
                case OpCode.setWatches:
                    zks.sessionTracker.checkSession(request.sessionId,
                            request.getOwner());
                    break;


来看pRequest2Txn，以create为例


    
      pRequest2Txn(request.type, zks.getNextZxid(), request, createRequest, true);
     
       protected void pRequest2Txn(int type, long zxid, Request request, Record record, boolean deserialize)
            throws KeeperException, IOException, RequestProcessorException
        {
            request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid,
                                        zks.getTime(), type);
    
            switch (type) {
                case OpCode.create:                
                    zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
                    CreateRequest createRequest = (CreateRequest)record;   
                    if(deserialize)
                        ByteBufferInputStream.byteBuffer2Record(request.request, createRequest);
                    String path = createRequest.getPath();
                    int lastSlash = path.lastIndexOf('/');
                    if (lastSlash == -1 || path.indexOf('\0') != -1 || failCreate) {
                        LOG.info("Invalid path " + path + " with session 0x" +
                                Long.toHexString(request.sessionId));
                        throw new KeeperException.BadArgumentsException(path);
                    }
                    List<ACL> listACL = removeDuplicates(createRequest.getAcl());
                    if (!fixupACL(request.authInfo, listACL)) {
                        throw new KeeperException.InvalidACLException(path);
                    }
                    String parentPath = path.substring(0, lastSlash);
                    ChangeRecord parentRecord = getRecordForPath(parentPath);
    
                    checkACL(zks, parentRecord.acl, ZooDefs.Perms.CREATE,
                            request.authInfo);
                    int parentCVersion = parentRecord.stat.getCversion();
                    CreateMode createMode =
                        CreateMode.fromFlag(createRequest.getFlags());
                    if (createMode.isSequential()) {
                        path = path + String.format(Locale.ENGLISH, "%010d", parentCVersion);
                    }
                    validatePath(path, request.sessionId);
                    try {
                        if (getRecordForPath(path) != null) {
                            throw new KeeperException.NodeExistsException(path);
                        }
                    } catch (KeeperException.NoNodeException e) {
                        // ignore this one
                    }
                    boolean ephemeralParent = parentRecord.stat.getEphemeralOwner() != 0;
                    if (ephemeralParent) {
                        throw new KeeperException.NoChildrenForEphemeralsException(path);
                    }
                    int newCversion = parentRecord.stat.getCversion()+1;
                    request.txn = new CreateTxn(path, createRequest.getData(),
                            listACL,
                            createMode.isEphemeral(), newCversion);
                    StatPersisted s = new StatPersisted();
                    if (createMode.isEphemeral()) {
                        s.setEphemeralOwner(request.sessionId);
                    }
                    parentRecord = parentRecord.duplicate(request.hdr.getZxid());
                    parentRecord.childCount++;
                    parentRecord.stat.setCversion(newCversion);
                    addChangeRecord(parentRecord);
                    addChangeRecord(new ChangeRecord(request.hdr.getZxid(), path, s,
                            0, listACL));
                    break;


- 首先是 zks.getNextZxid()创建一个事务id，AtomicLong hzxid是自增长id，初始化为0，每次加一
- 在pRequest2Txn内部，先给request创建一个TxnHeader，这个header包含事务id
- 然后判断请求类型
- zks.sessionTracker.checkSession(request.sessionId, request.getOwner()) 检查session
- 反序列化为CreateRequest


##### SyncRequestProcessor

##### FinalRequestProcessor

未完待续

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


