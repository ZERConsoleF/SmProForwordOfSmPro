这个文档是会如何教你使用它的

# 编写语言:
   C#4.7
# 在Linux和Mac等系统上运行需支持库
   mono 6.8.x
# 更新内容
   解决了内存转发导致的内存空间不够  
   解决了名字显示问题  

# 说明:
    工具:  
      Client:  
         关于Forword转发客户端  
      Server:  
         关于Forword转发客户端的连接点  
      CreativeCAI:  
         创建连接服务点的证书  
   参数: 
   
      work.ini  
         IPAdress:创建连接点或制造连接点的IP  
         Port:创建或连接的端口  
         CAIConnent:CAI连接证书(客户端需要生成CAI证书,服务器随便)  
         Index:制造或验证的指纹  
         LogPath:生成日志的文件夹  
         SendSleep:等待发送列队发送时间(越小发送的延迟越小,同时CPU的占用就高)  

         客户端独有:ForwordPort:添加转发列表  
            ForwordIPAdress:转发连接IP  
            LocalPort:本地连接端口  
            RemotePort:远程映射端口  
            PassbyPort:转发通道  
            ReTitleGet:分配转发时用到的byte字节分配  
# 如何实现:

    先开启服务器,然后客户端会把添加列表发送给服务器,然后服务器会通过添加列表开启对应的端口和通道  
    然后客户端会连接并创建2个通道(作用就是发送转发数据和接收数据)  
    等待用户连接  
    当第一个用户连接时  
    服务器会通过通道发送空字节来激活转发通道  
    客户端通过这个添加转发清单来对应连接本地服务器(如果连接失败或其他原因,客户端会给服务器发送2消息代码来断开连接)  
    之后这个连接者会分别保存在Sockets列队中,同时会开启一个Task来监听连接服务器和用户的发送请求  

如何转发: 
   
      首先当监听列队(客户端或服务器,下同)收到一个请求时,它不会马上发送给客户端(或服务器),而是会添加到"sendlist"发送列队中  
      之后系统会判断"sendlist"发送列队中是否有数据  
      如果有,启动通道并发送发送列队的数据,没有继续等待(因为等待是通过while实现的,为了避免CPU浪费,所以会添加"等待发送列队")  
   
如何接受并转发给连接服务器(或用户):  
      监听列队收到一个请求时,并不会立刻添加到发送列队中,而是会在数据头会添加字节长度为24的"头"  
      这个头主要包含:  
      
         用户IP,转发代码  
      
然后会把正文添加在字节数为24之后  
      
      当客户端(服务器)接受到这个请求,首先会解析,把"头"解析到,然后解析正文  
      首先会先判断用户IP,如果有匹配项,则调用这个Socket  
      然后会解析转发代码:  
         通常这些代码表达的意思:  
            0:没问题,转发  
            1:我这边在接收监听列表数据出现了问题,我这边把它断开了,你需要把它断开  
            2:没问题,但是这可能是激活数据,所以你不需要转发数据正文  
            3(现在为none):这个数据是用来维持通道的,你不需要解释  
当然接收和转发数据会启动Task任务  
   我是如何维持通道:  
   
      大家都知道,有些路由如果长时间不发送数据,它会关闭这个通道  
      所以我是在(在服务器中)启动Accept接收用户时,开启Task后台线程  
      这个线程主要是用来定时添加空数据到发送列表  
      我设置的时间为:1分钟  
# 这个主要的问题和可能的解决:
   ftp出现:无法与服务器建立连接  
      这个问题主要是因为ftp它发送比较急,会导致发送列表发送完后进入等待  
      此时又有数据发送进来,但是ftp限定的接收延迟比较小,会导致以为没有请求,ftp会再次发送请求  
      这将会导致重复发送,最终数据重复,ftp解析失败  
   解决:把延迟调小或不要延迟发送  
   
   有时候重新添加则无法开启远程端口:  
   解决:关了重启软件  
