### .Net Grpc使用Consul注册发现服务和服务调用

#### 一、Grpc 服务编写

- 新建Grpc服务项目 
    
    - Grpc服务根据定义的Proto文件生成对应的C#类，前提是定义完后在项目文件里添加文件引用

        - 编辑项目文件，xxx.csproj，添加项

        - Protobuf表示文件类型是Proto

        - Include 表示引入的文件地址

        - GrpcServices 表示服务类型 Server表示是服务端，如果定义客户端需要改为Client，如果都支持就设置为Both，表示既是客户端又是服务端

        - 访问Grpc服务必须使用Grpc Client

        ```xml
          <ItemGroup>
            <Protobuf Include="Protos\greet.proto" GrpcServices="Server" />
          </ItemGroup>
        ```
    - 项目文件里添加引用后，生成项目，会将Proto编译成的c#类文件生成在项目文件夹 /obj/debug/net6.0下。

    - 编写Service类，继承Proto文件生成的类的基类，如

        ```csharp
        public class GreeterService : Greeter.GreeterBase
        {
            private readonly ILogger<GreeterService> _logger;
            public GreeterService(ILogger<GreeterService> logger)
            {
                _logger = logger;
            }
    
            public override Task<HelloReply> SayHello(HelloRequest request, ServerCallContext context)
            {
                return Task.FromResult(new HelloReply
                {
                    Message = "Hello " + request.Name
                });
            }
        }
        
        ```


- 引入包
        
    -  $\color{red}{Grpc.Aspnetcore:}$ Grpc aspnetcore 支持包
        
    -  $\color{red}{Grpc.HealthCheck:}$ Grpc 健康检查

    -  $\color{red}{NConsul.Aspnetcore:}$ 支持Grpc健康检查和便捷添加服务的Consul组件包，封装原生注册consul服务的方法，且直接支持健康检查。作者微软VIP晓晨Master.
        
    -  $\color{Indigo}{想使用原生组件的化，下面在健康检查的介绍里会介绍原生Consul组件的调用和健康检查方法}$ 

- 添加服务 比较简单的方式
    - .Net6没有startup类，所有入口的类都在program下

    - 路由处官方demo给的备注，访问Grpc服务必须使用Grpc Client

    - AddGrpc 把Grpc服务添加到容器

    - AddConsul 就是NConsul组件封装的，添加Consul服务，参数是Consul的地址

    - AddGRPCHealthCheck 添加健康检查服务，参数是服务地址，是否ssl,超时秒，间隔秒

    - RegisterService就是把GRPC服务注册到Consul，参数是服务名称，地址，端口，Tags，这样Consul就会发现服务和自动伸缩了（就是你新服务通过检查后会被加进来，健康检查失败的服务会被T掉）

    -  app.MapGrpcService<GreeterService>(); 把你自己的服务类添加到管道

    -  app.MapGet("/") 配置请求端点

    ```csharp
        using GrpcConsulTest.Services;
        using NConsul.AspNetCore;
        
        var builder = WebApplication.CreateBuilder(args);
        // Add services to the container.
        builder.Services.AddGrpc();
        builder.Services.AddConsul("http://localhost:8500")
             .AddGRPCHealthCheck("localhost:5266", true, 10, 10)
             .RegisterService("grpctest", "localhost", 5266, new[] { "xc/grpc/test" });
        var app = builder.Build();
        
        // Configure the HTTP request pipeline.
        app.MapGrpcService<GreeterService>();
        app.MapGrpcService<LucatService>();
        app.MapGrpcService<HealthCheckService>();
        app.MapGet("/", () => "Communication with gRPC endpoints must be made through a gRPC client. To learn how to create a client, visit: https://go.microsoft.com/fwlink/?linkid=2086909");
        
        app.Run();
       
- 添加服务，使用Consul官方组件：

     ```csharp
             var consul = new ConsulClient(x =>
                    {
                        x.Address = new Uri(options.ConsulUrl);
                    });
        
        
                    var registration = new AgentServiceRegistration
                    {
                        ID = Guid.NewGuid().ToString(),//服务实例唯一标识
                        Name = options.ServiceName,//服务名称
                        Address = options.ServiceHost,//服务IP地址
                        Port = options.ServicePort,
                        Check = new AgentServiceCheck//健康检查
                        {
                            DeregisterCriticalServiceAfter = TimeSpan.FromSeconds(5),//有人说这个是标识服务启动多久后注册，但是字面意思好像是取消注册,应该是健康检查多久没通过取消注册
                            Interval = TimeSpan.FromSeconds(5),//间隔时间
                            Timeout = TimeSpan.FromSeconds(5),//超时时间
                            GRPC = options.HealthCheckHost,//Grpc必须使用Grpc
                            GRPCUseTLS = options.Usetls，//是否使用tls
                        }
                    };
        
                    consul.Agent.ServiceRegister(registration);//服务注册
                    //程序终止取消注册
                    lifetime.ApplicationStopping.Register(() =>
                    {
                        consul.Agent.ServiceDeregister(registration.ID).Wait();
                    });

    ``` 
    
- 新建Grpc客户端

    - 引用包
        - $\color{red}{Grpc.Net.Client:}$ Grpc .Net客户端

        - $\color{red}{Grpc.Tools:}$ Grpc 把proto文件编译为c#代码的依赖包

        - $\color{red}{Google.Protobuf:}$ 狗哥官方的Protobuf C#运行时库

        - 以上三个包是C# Grpc客户端程序必须引用的包，引用后即可创建Client调用Grpc服务

        - $\color{red}{Consul:}$ Consul组件，允许.net程序使用Consul

    - 修改项目文件

        - 因为grpc依赖Proto文件，client端也需要和服务端一样的proto，这样就要维护两份一模一样的proto，比较省事的办法就是把proto放在一个公用的路径，在项目文件里统一引用共同的proto文件。
        
        - 在实际项目中使用，肯定有多个 proto 文件，每次添加一个Proto文件就要更新一次csproj文件，避免这样的麻烦，可以使用MSBuild来帮助完成文件添加，服务端和客户端一样，只需要改变GrpcServices的值即可：
        
             ```xml
              <ItemGroup>
               <Protobuf Include="..\Protos\*.proto" GrpcServices="Server"     
                         Link="Protos\%(RecursiveDir)%(Filename)%(Extension)" />            
              </ItemGroup>
            ```
        
        - 直接调用Grpc，需要创建一个channel连接服务地址，再创建client调用方法，如
             ```csharp
                var channel = GrpcChannel.ForAddress("https://localhost:5001");
                var client = new Greeter.GreeterClient(channel);
                var reply = await client.SayHelloAsync(new HelloRequest { Name = "HH" });
                Console.WriteLine("Greeter 服务返回数据: " + reply.Message);
                Console.ReadKey();
            ```

        
#### 二、本地开发使用Consul调试

- Consul的概念

    - 这里只简述：Consul是使用Go语言编写的，轻量级的服务发现和注册组件，支持多种编程语言，功能包括服务发现和注册，健康检查，弹性伸缩，KeyValue配置，支持集群部署

- Consul安装

    - Consul安装方式有多种，生产环境基本使用Docker部署集群， 使用Docker拉服务方便水平扩展

    - 开发环境可以使用exe文件也可以使用docker拉取镜像，因为电脑性能影响，我这里直接使用本地运行
        - 官网下载 consul.exe

        - 放在你想放的位置，添加系统变量或者切换到你的文件夹，执行Consul agent -dev 就运行起来一个Consul节点了。默认的访问端口是8500,所以你访问http://127.0.0.1:8500/会看到consul的ui界面。

- 运行服务：启动上面的Grpc服务，你会看到自己注册的服务显示在界面上了，断点健康检查的方法，会发现Consul自动在调用健康检查的方法
    - 如果健康检查正常，需要返回在线的状态，不然consul会检查失败，HealthCheckService：
    
     ```csharp
        public class HealthCheckService : HealthServiceImpl
            {
                public override Task<HealthCheckResponse> Check(HealthCheckRequest request,ServerCallContext context)
                    {
                        return Task.FromResult(new HealthCheckResponse() { Status = HealthCheckResponse.Types.ServingStatus.Serving });
                    }
        
                public override async Task Watch(HealthCheckRequest request, IServerStreamWriter<HealthCheckResponse> responseStream, ServerCallContext context)
                    {
                        await responseStream.WriteAsync(new HealthCheckResponse()
                        { Status = HealthCheckResponse.Types.ServingStatus.Serving });
                    }
            }
    ```
    

#### 三、服务的调用

- 上面需要的基础组件和项目搭好后就可以使用客户端来调用Consul中的Grpc服务了

- 首先，创建Consul连接，连接到Consul服务：

    ```csharp
    var consulClient = new ConsulClient(c => c.Address = new Uri("http://localhost:8500"));
    ```
- 获取服务：

    ```csharp
        var serviceName="Test";
        var services = await consulClient.Catalog.Service(serviceName);
        if (services.Response.Length == 0)
            {
                throw new Exception($"未发现服务 {serviceName}");
            }
        var service = services.Response[0];
    ```

- 访问服务执行调用：

    ```csharp

        //获取服务地址
        var address = $"http://{service.ServiceAddress}:{service.ServicePort}";
        
        Console.WriteLine($"获取服务地址成功：{address}");
        
        //启用通过http使用http2.0
        AppContext.SetSwitch(
            "System.Net.Http.SocketsHttpHandler.Http2UnencryptedSupport", true);
        //创建Grpc连接
        var channel = GrpcChannel.ForAddress(address);
        var greeterClient = new Greeter.GreeterClient(channel);
        var greeterReply = await greeterClient.SayHelloAsync(new HelloRequest { Name="HH"});
        Console.WriteLine("调用打招呼服务：" + greeterReply.Message);
        Console.ReadKey();
        
    ```

#### 四、Consul动态负载

- 目前的有多种方案，最优的应该是nginx+upsync+consul，这个具体的实现我这里省略了，参考
[CSDN](https://blog.csdn.net/weixin_54931703/article/details/120874717?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_default&utm_relevant_index=2)
[CNBLOG](https://www.cnblogs.com/haoworld/p/nginx-ji-yuconsulupsyncnginx-shi-xian-dong-tai-fu-.html)

至此 Grpc服务和Consul注册发现及调用就完成了，更多更详细的内容请访问微软官网或者博客园晓晨Master大佬的教程。当然或许还有更便捷的内容，后期会再添加进本篇文章内。

[微软文档](https://docs.microsoft.com/zh-cn/aspnet/core/grpc/?view=aspnetcore-3.0&WT.mc_id=DT-MVP-5003133 '.Net Grpc')

[晓晨Master的Grpc教程](https://www.cnblogs.com/stulzq/p/11897704.html 'cnblogs')

[Demo](https://gitee.com/lionkon/net-grpc-consul-demo "Demo")