Protobuf(Protocol Buffer)是一种数据通信协议，相比JSON，它的传输数据量更小，而且没有对应的proto文件根本无法看懂传输的二进制格式数据，所以传输安全性也更高。
> Protobuf的安装就不多做介绍了，这是之前写的[Protobuf安装踩坑记录](http://blog.sina.com.cn/s/blog_942d71bb0102wswn.html) 
放一个批量编译.proto文件的命令：
批量编译Module目录下的所有proto文件并输出`protoc -I=./Module --objc_out=./Module ./Module/*.proto`

这里主要列出使用其通信时需要处理的情况及常规写法，根据不同公司架构自行调整，下图是与后台协商后的数据包装格式，来源于游戏架构。
![字很丑=_=不要在意这些细节](http://upload-images.jianshu.io/upload_images/4483590-96de8b38f4405c26.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# HTTP短连接
### Request格式
|  字段  |  长度  |  类型  |  数值  |
| :---: | :---: | :---: | :---: |
| 0xff | 4 | `int`（`int32_t`） | 大端255，小端-1677216 |
| playerId | 8 | `long`（`int64_t`） | 用户id |
| sessionId | 8 | `long`（`int64_t`） | 会话id |
| size | 4 | `int`（`int32_t`） | data.length+4 |
| commandId | 4 | `int`（`int32_t`） | 协议号id |
| data | data.length | `NSData` | 数据 |

### Response格式
|  字段  |  长度  |  类型  |  数值  |
| :---: | :---: | :---: | :---: |
| 0xff | 4 | `int`（`int32_t`） | 大端255，小端-1677216 |
| size | 4 | `int`（`int32_t`） | data.length+4 |
| commandId | 4 | `int`（`int32_t`） | 协议号 |
| data | data.length | `NSData` | 数据 |

#### AFNetworking 3.0+Protobuf
> 这里封装了一个网络访问的方法，要注意的是大小端处理，即苹果系统在传输前要先将整数类型数据转为小端序

    - (void)requestProtobufCommandId:(int)commandId
                              params:(NSData *)params
                            playerId:(uint64_t)playerID
                           sessionId:(uint64_t)sessionID
                   completionHandler:(void(^)(NSData *data))dataBlock
                        errorHandler:(void(^)(int32_t errorCode))errorBlock {
    
    // 苹果 整数字节使用小端序传输，而其他都是网络端序大端序传输
    // HTONS 转换端序，从小端序转为大端序，HTONL 转化4字节端序，HTONLL转化8字节端序。
    // int  htonl ->short 类型的主机存储－>网络的网络存储，并写入内存块
    // char,string 类型不需要转换
    
    NSMutableData *protobufData = [[NSMutableData alloc] init];
    // 0XFF
    int str = 0xff;
    str = htonl(str);
    [protobufData appendBytes:&str length:sizeof(str)];
    // playerId
    uint64_t playerId = 0;
    playerId = htonll(playerId);
    NSData *playerIdData = [NSData dataWithBytes: &playerId length: sizeof(playerId)];
    [protobufData appendData:playerIdData];
    // sessionId
    uint64_t sessionId = 0;
    sessionId = htonll(sessionId);
    NSData *sessionIdData = [NSData dataWithBytes: &sessionId length: sizeof(sessionId)];
    [protobufData appendData:sessionIdData];
    // size
    u_long size = params.length+4;
    size = htonl(size);
    [protobufData appendBytes:&size length:4];
    // commandId
    commandId = htonl(commandId);
    NSData *commandIdData = [NSData dataWithBytes: &commandId length: sizeof(commandId)];
    [protobufData appendData:commandIdData];
    // data
    [protobufData appendData:params];
    
    Byte *byte = (Byte *)[protobufData bytes];
    NSString *byteString = @"";
    for (int i=0 ; i<[protobufData length]; i++) {
        byteString = [byteString stringByAppendingString:[NSString stringWithFormat:@"%d ",byte[i]]];
    }
    NSLog(@"byteString: %@",byteString);
    
    //第一步，创建url
    NSURL *url = [NSURL URLWithString:@"192.168.1.1:8080"];
    //第二步，创建请求
    NSMutableURLRequest *request = [[NSMutableURLRequest alloc]initWithURL:url cachePolicy:NSURLRequestUseProtocolCachePolicy timeoutInterval:10];
    [request setHTTPMethod:@"POST"];
    [request setHTTPBody:protobufData];
    //第三步，连接服务器
    AFHTTPSessionManager *_manager = [AFHTTPSessionManager manager];
    //不可使用JSON序列化方式
    _manager.responseSerializer = [AFHTTPResponseSerializer serializer];
    
    NSURLSessionDataTask *task = [_manager dataTaskWithRequest:request completionHandler:^(NSURLResponse * _Nonnull response, id  _Nullable responseObject, NSError * _Nullable error) {
        if (error){
            NSLog(@"error = %@",error);
            dataBlock(nil);
        }else{
            NSLog(@"protobuf data = %@", responseObject);
            
            NSData *data = responseObject;
            if (data.length > 12) {
                // normal data
                dataBlock([data subdataWithRange:NSMakeRange(12, data.length-12)]);
            } else {
                dataBlock(nil);
            }
        }
    }];
    [task resume];
    }
    
##### 使用方法如下
    Test_Req *req = [Test_Req new];
    req.param1 = @"0";
    req.param2 = 1;
    [self requestProtobufCommandId:CommandEnum_CmdTest params:[req data] playerId:0 sessionId:0 completionHandler:^(NSData *data) {
        if (data) {
            Test_Rsp *rsp = [Test_Rsp parseFromData:data error:nil];
            NSLog(@"rsp: %@, result1:%@, result2: %lld", rsp,rsp.result1,rsp.result2);
        }
    } errorHandler:^(int32_t errorCode) {
        NSLog(@"errorCode: %d",errorCode);
    }];
这里要注意必须使用Req对应的Rsp去解析返回的数据，如果协议号不相同则会解析失败，得到的只会是一段乱码


# Socket长连接
###### 约定的Request与Response格式是一样的
|  字段  |  长度  |  类型  |  数值  |
| :---: | :---: | :---: | :---: |
| 0xff | 4 | `int`（`int32_t`） | 大端255，小端-1677216 |
| size | 4 | `int`（`int32_t`） | data.length+4 |
| commandId | 4 | `int`（`int32_t`） | 协议号 |
| data | data.length | `NSData` | 数据 |

#### CocoaAsyncSocket+TCP+Protobuf
这里使用的是CocoaAsyncSocket，并提供了二次封装类的写法。

    //初始化socket并连接到服务端
    - (void)initSocket {
        GCDAsyncSocket *gcdSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
    }

    #pragma mark - 对外的一些接口
    //建立连接
    - (BOOL)connect {
        return [gcdSocket connectToHost:@"192.168.1.1" onPort:8080 error:nil];
    }

    //断开连接
    - (void)disConnect {
        [gcdSocket disconnect];
    }

    #pragma mark - GCDAsyncSocketDelegate
    //连接成功调用
    - (void)socket:(GCDAsyncSocket *)sock didConnectToHost:(NSString *)host port:(uint16_t)port {
        NSLog(@"连接成功,host:%@,port:%d",host,port);
        //监听读数据的代理  -1永远监听，不超时，但是只收一次消息，
        //所以每次接受到消息还得调用一次
        [gcdSocket readDataWithTimeout:-1 tag:0];
        //心跳写在这...
    }

    //断开连接的时候调用
    - (void)socketDidDisconnect:(GCDAsyncSocket *)sock withError:(nullable NSError *)err {
        NSLog(@"断开连接,host:%@,port:%d",sock.localHost,sock.localPort);
        //断线重连写在这...
        if (err) {
            //再次重连
        } else {
          //正常断开
        }
    }

CocoaAsyncSocket提供了读数据和写数据的方法
- `[gcdSocket readDataWithTimeout:-1 tag:0];` 用来收取消息，-1表示永远不超时，
- `[gcdSocket writeData:protobufData withTimeout:-1 tag:0];`用来写入数据，将数据按要求转化为protobuf data即可

发送协议的方法如下
    
    //发送数据流和协议号
    - (void)sendProbufData:(NSData *)data
                 CommandId:(int)commandId {
    NSMutableData *protobufData = [[NSMutableData alloc] init];
    // 0XFF
    int str = 0xff;
    str = htonl(str);
    [protobufData appendBytes:&str length:sizeof(str)];
    // size
    u_long size = data.length+4;
    size = htonl(size);
    [protobufData appendBytes:&size length:4];
    // commandId
    commandId = htonl(commandId);
    NSData *commandIdData = [NSData dataWithBytes: &commandId length: sizeof(commandId)];
    [protobufData appendData:commandIdData];
    // data
    [protobufData appendData:data];

    Byte *byte = (Byte *)[protobufData bytes];
    NSString *byteString = @"";
    for (int i=0 ; i<[protobufData length]; i++) {
        byteString = [byteString stringByAppendingString:[NSString stringWithFormat:@"%d ",byte[i]]];
    }
    NSLog(@"byteString: %@",byteString);
    // 写入数据
    [gcdSocket writeData:protobufData withTimeout:-1 tag:0];
    }

尝试向服务端发送一个心跳包，写法如下

     - (void)heartBeat {
        //生成一个空的请求占位
        Empty_Req *req = [Empty_Req new];
        [self sendProbufData:[req data] CommandId:CommandEnum_CmdHeartBeat];
    }
收到消息处理数据

    //收到消息的回调
    - (void)socket:(GCDAsyncSocket *)sock didReadData:(NSData *)data withTag:(long)tag {
        NSLog(@"收到消息：%@",data);
        if (data.length > 12) {
            NSData *cmdIdData = [data subdataWithRange:NSMakeRange(8, 4)];
            int j; // j为协议号的数值
            [cmdIdData getBytes: &j length: sizeof(j)];
            j = htonl(j);
        
            NSData *sizeData = [data subdataWithRange:NSMakeRange(4, 4)];
            int i;  //i为协议号与数据流字节的长度之和，在有拼接数据时使用
            [sizeData getBytes: &i length: sizeof(i)];
            i = htonl(i);

            if (j == CommandEnum_CmdHeartBeat) {
                // 心跳包不做处理
                NSLog(@"收到了心跳包!");
            } else {
                // 这里解析正常数据...
            }
        }
        //继续接收数据
        [gcdSocket readDataWithTimeout:-1 tag:0];
    }

# 地址
> @黑花白花 的博文[Runtime在实际开发中的应用](http://www.jianshu.com/p/851b21870d91)第一节提供了一个Protobuf解析器，帮助PB编译出Model与普通Model之间相互转换：[PBPaser](https://github.com/HeiHuaBaiHua/TRuntime/tree/master/ProtobufParser)

[完整Demo地址](https://github.com/HelloiWorld/UseProtobufDemo.git)
