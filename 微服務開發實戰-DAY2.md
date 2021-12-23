# 微服務開發實戰

## 安裝.Net Core 開發環境

* 下載.Net Core SDK

  * Windows https://dotnet.microsoft.com/download/dotnet/thank-you/sdk-5.0.403-windows-x64-installer

  * Mac https://dotnet.microsoft.com/download/dotnet/thank-you/sdk-5.0.403-macos-x64-installer

  * Ubuntu 

    新增套件存放庫

    ``` bash
    wget https://packages.microsoft.com/config/ubuntu/21.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
    sudo dpkg -i packages-microsoft-prod.deb
    ```

    安裝SDK

    ```bash
    sudo apt-get update; \
      sudo apt-get install -y apt-transport-https && \
      sudo apt-get update && \
      sudo apt-get install -y dotnet-sdk-5.0
    ```

    安裝Runtime

    ```bash
    sudo apt-get update; \
      sudo apt-get install -y apt-transport-https && \
      sudo apt-get update && \
      sudo apt-get install -y aspnetcore-runtime-5.0
    ```

    Linux其他版本安裝方式

    https://docs.microsoft.com/zh-tw/dotnet/core/install/linux?WT.mc_id=dotnet-35129-website

* 檢查安裝版本  

  * 列出已安裝的SDK

    ```ba
    dotnet --list-sdks
    ```

  * 列出已安裝的Runtime

    ```ba
    dotnet --list-runtimes
    ```

* 開發工具

  建議開發使用Visual Studio Code

  安裝路徑 https://code.visualstudio.com/download 

  開發安裝外掛，

  https://marketplace.visualstudio.com/items?itemName=doggy8088.netcore-extension-pack

  (這是⼀個整合包，不是單一包，就是保哥把所有相關的套件都集合在一起，不⽤一個⼀個裝)
  
* 安裝Node.js

  安裝網址 https://nodejs.org/zh-tw/download/
  
* 安裝Insomnia (類似Postman，介面比較友善)

  安裝網址 https://insomnia.rest/download

## 樣板專案

* 安裝Yeoman

  ```bash
  npm install -g yo
  ```

* 安裝示範專案產生器

  ```bash
  npm install -g generator-himsk8s
  ```

* 建立專案目錄

  ```bash
  mkdir k8sTestProject
  ```

* 產生專案

  ```bash
  yo himsk8s
  ```

  請直先按下Enter鍵或是選擇Y，變會自動產生專案，結果如下：

  其中『微服務開發實戰.md』就是今天教學的講義。

  ```bash
  ├───CommonService/
  │   └───CommonService.csproj
  ├───微服務開發實戰.md
  └───himsk8s.sln
  ```

  推薦用Typora開啟，Typora下載網址如下：(目前Beta版不用收費)

  MacOS的下載網址：https://typora.io/dev_release.html

  Winsows及Linux的下載網址：https://typora.io/windows/dev_release.html

* 產生微服務專案

  ```bash
  yo himsk8s:service
  ? Enter the new service name TestService
  ? Enter the new service's port (9000) 
  ? Enter the new service's ssl port (9001) 
  ```

  最後產出結果應該如下：

  增加兩個目錄：depls（K8S的Deployment file）、TestService（新產生出來的微服務專案）

  ```bash
  ├───CommonService/
  │   └───CommonService.csproj
  ├───depls/
  ├───TestService/
  │   └───Controllers/
  │   		└───QuerySampleController.cs
  │   └───Services/
  │   		└───QuerySampleData/
  │   				└───IQuerySampleData.cs
  │   				└───QuerySampleData.cs
  ├───微服務開發實戰.md
  └───himsk8s.sln
  ```

  預設微服務專案中有一個簡單的資料庫查詢範例QuerySampleController，連結一個簡單服務QuerySampleData

* QuerySampleData

  * IQuerySampleData.cs 介面

    ```c#
    using CommonService.Dtos;
    using System.Data;
    
    namespace TestService.Services
    {
        public interface IQuerySampleData
        {
             ResultDTO<DataSet> QuerySampleDataFromDB();
        }
    }
    ```

  * QuerySampleData.cs 實作

    ```c#
    using System;
    using System.Threading.Tasks;
    using System.Data;
    using CommonService.Utilities;
    using CommonService.Dtos;
    using CommonService.Params;
    using Microsoft.Extensions.Configuration;
    using Newtonsoft.Json;
    
    namespace TestService.Services
    {
        public class QuerySampleData: IQuerySampleData
        {         
            private readonly IConfiguration _config;        
    
            public QuerySampleData(IConfiguration configuration)
            {
                Console.WriteLine("QueryData Init");
                _config = configuration;
            }
    
    
            public ResultDTO<DataSet> QuerySampleDataFromDB()
            {
                Console.WriteLine("QuerySampleDataFromDB Init");
                string message = "";
                string messageLevel = "";
                string messageType = "";
                DataSet queryDS = new DataSet();
                try
                {
                    BaseParams param = new BaseParams();
                    string[] serverInfo = _config.GetSection("AppSettings")["SQLString"].Trim().Split('|');
                    param.SERVER_NAME = serverInfo[0];
                    param.DATABASE_NAME = serverInfo[1];
                    param.LOGIN = serverInfo[2];
                    param.PASSWORD = serverInfo[3];
                    param.DATABASE_TYPE = "S";
                    param.SSAPI_ADDRESS = _config.GetSection("AppSettings")["SSAPI"].Trim();
    
                    object[][] queryParam = {
                            new object[1]  { "" }
                       };
                    serverInfo=new ServerInfo(_config).GetServerInfo(param);
                    
                    Console.WriteLine("WebApiClient Init" + JsonConvert.SerializeObject(serverInfo));
                    queryDS = new WebApiClient(_config).SPExeBatchMultiArr2(serverInfo, "SP_TEST010_Q", queryParam, false , ref message, ref messageLevel, ref messageType);
    
                    Console.WriteLine("WebApiClient After"+queryDS.Tables[0].Rows.Count.ToString());
                    if (message != "")
                    {
                        throw new Exception(message + "<br>" + JsonConvert.SerializeObject(queryParam));
                    }
    
                    return (new ResultDTO<DataSet>
                    {
                        Message = "",
                        Data = queryDS,
                        Result = true
                    });
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"--> QueryData has error: {ex.Message.Trim()}");
                    return (new ResultDTO<DataSet>
                    {
                        Message = ex.Message,
                        Data = null,
                        Result = false
                    });
                }
    
            }
    
            
        }
    }
    
    ```

 * 連線位置及參數：

   appsettings.json(正式環境設定)、appsettings.Development.json(測試環境設定)
   
   * appsettings.json
   
     ```json
     {
       "Logging": {
         "LogLevel": {
           "Default": "Information",
           "Microsoft": "Warning",
           "Microsoft.Hosting.Lifetime": "Information"
         }
       },
       "AppSettings":{
         "SERVER_NAME":"SSFORMOSA",	//預設連線主機
         "DATABASE_NAME":"HTNS",	//預設連線資料庫
         "LOGIN":"",	//預設連線帳號
         "PASSWORD":"",	//預設連線密碼
         "DATABASE_TYPE":"S",
         "SSAPI":"https://ap.searching-service.com/ssapi45/",	//設定連線的SSAPI
         "SQLString":"SSFORMOSA|HTNS||"	//連線主機、資料庫、帳號、密碼，用『|』進行分隔
       }
     }
     ```

* 測試執行

  * DotNet Cli

    切換到該專案目錄下方，執行以下指令

    ```bash
    dotnet run
    ```

  * Debugger

    設定.vscide/launch.json的偵錯設定進行除錯，可以設定中斷點，

    增加以下設定區段到configurations陣列中，置換掉TestService字眼，改為你所產生的專案名稱

    ```json
    {
                // Use IntelliSense to find out which attributes exist for C# debugging
                // Use hover for the description of the existing attributes
                // For further information visit https://github.com/OmniSharp/omnisharp-vscode/blob/master/debugger-launchjson.md
                "name": "TestService",
                "type": "coreclr",
                "request": "launch",
                "preLaunchTask": "build",
                // If you have changed target frameworks, make sure to update the program path.
                "program": "${workspaceFolder}/TestService/bin/Debug/net5.0/TestService.dll",
                "args": [],
                "cwd": "${workspaceFolder}/TestService",
                "stopAtEntry": false,
                // Enable launching a web browser when ASP.NET Core starts. For more information: https://aka.ms/VSCode-CS-LaunchJson-WebBrowser
                "serverReadyAction": {
                    "action": "openExternally",
                    "pattern": "\\bNow listening on:\\s+(https?://\\S+)"
                },
                "env": {
                    "ASPNETCORE_ENVIRONMENT": "Development"
                },
                "sourceFileMap": {
                    "/Views": "${workspaceFolder}/Views"
                }
            }
    ```

    設定完成之後，可以到VSCode的執行與偵錯中進行偵錯的動作

## 建立Docker Image

* 專案檔案結構

```
├───CommonService/
│   └───CommonService.csproj
├───QueryService/
│   └───QueryService.csproj
└───himsk8s.sln
```

* Dockerfile內容

```yaml
FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build-env
WORKDIR /app

COPY QueryService/*.csproj QueryService/
COPY CommonService/*.csproj CommonService/
RUN dotnet restore "QueryService/QueryService.csproj"

COPY . ./
RUN dotnet publish "QueryService/QueryService.csproj" -c Release -o out

FROM mcr.microsoft.com/dotnet/aspnet:5.0
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT [ "dotnet","QueryService.dll" ]
```

* 建立Docker Image

  因爲建立Docker Image要引用共用專案，所以執行CLI的時候要在專案的根目錄下執行

```bash
docker build -f QueryService/Dockerfile -t jihlihuang/queryservice .
```

* 註冊Image並掛載Container執行

  ```bash
  docker run -p 8080:80 jihlihuang/queryservice
  ```

  可以連線的網址為：http://localhost:8080/api/QuerySampleData

* Docker 常用指令

```bash
#註冊到Docker服務中
docker run -p 8080:80 jihlihuang/queryservice
#檢視目前有多少已註冊的image
docker ps
#啟動Docker image
docker start jihlihuang/queryservice
#停止Docker image
docker stop jihlihuang/queryservice
#移除docker container
docker rm <Container_ID>
#移除docker image
docker image rm jihlihuang/queryservice
```

* Push to Docker Hub

直接推送到Docker Hub，如果要推送到其他的私有庫，可以後方接著私有庫的位置

``` bash
docker push jihlihuang/queryservice
```

## 部署至K8S

離開原本的專案目錄，到上層目錄中，建立部署檔案的目錄K8S

* 建立deployment的yaml檔案(queryservice-depl.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queryservice-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: queryservice
  template:
    metadata:
      labels:
        app: queryservice
    spec:
      containers:
        - name: queryservice
          image: jihlihuang/queryservice:latest
```

* 發布到k8s

```ba
kubectl apply -f queryservice-depl.yaml
```

* k8s相關指令

```bash
#發布及更新
kubectl apply -f queryservice-depl.yaml
#取得發布結果
kubectl get deployments
#取得pod清單
kubectl get pods
#移除發布，所有的pod都會被終結
kubectl delete deployments queryservice-depl
#重新由Docker Hub上更新K8S的服務，並且執行重啟
kubectl rollout restart deployment queryservice-depl
```

## 建立Node Port Forward

* 新增Service設定檔queryservice-np-srv.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: queryservice-np-srv
spec:
  type: NodePort
  selector:
    app: queryservice
  ports:
    - name: queryservice
      protocol: TCP
      port: 80
      targetPort: 80
```

* 發布

```ba
kubectl apply -f queryservice-np-srv.yaml
```

* 取得發布結果

```bash
kubectl get services
```

* 取得Node port 進行連線測試

```bash
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP        65d
queryservice-srv   NodePort    10.105.220.185   <none>        80:32453/TCP   29s
```

連線位置==>http://localhost:32453/api/QuerySampleData

## 建立Cluster-IP

更新發佈檔，增加ClusterIP的定義

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queryservice-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: queryservice
  template:
    metadata:
      labels:
        app: queryservice
    spec:
      containers:
        - name: queryservice
          image: jihlihuang/queryservice:latest
#ClusterIP定義區塊
---
apiVersion: v1
kind: Service
metadata:
  name: queryservice-clusterip-srv
spec:
  type: ClusterIP
  selector:
    app: queryservice
  ports:
  - name: queryservice
    protocol: TCP
    port: 80
    targetPort: 80
```

可以使用以下指令檢視是否queryservice-clusterip-srv的服務出現

```bash
kubectl get services
```

## 更新K8S的Service

重新產出新的Docker Image

```bash
docker build -f QueryService/Dockerfile -t jihlihuang/queryservice .
```

推送到Docker Hub

```bash
docker push jihlihuang/queryservice
```

執行K8S服務更新

```bash
kubectl rollout restart deployment queryservice-depl
```

## 建立RabbitMQ

* 安裝RabbitMQ至Kubenetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:3-management
          ports:
            - containerPort: 15672
              name: rbmq-mgmt-port
            - containerPort: 5672
              name: rbmq-msg-port
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-clusterip-srv
spec:
  type: ClusterIP
  selector:
    app: rabbitmq
  ports:
  - name: rbmq-mgmt-port
    protocol: TCP
    port: 15672
    targetPort: 15672
  - name: rbmq-msg-port
    protocol: TCP
    port: 5672
    targetPort: 5672
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: rabbitmq
  ports:
  - name: rbmq-mgmt-port
    protocol: TCP
    port: 15672
    targetPort: 15672
  - name: rbmq-msg-port
    protocol: TCP
    port: 5672
    targetPort: 5672
```

* 建立RabbitMQ訊息發布

```c#
using System;
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Configuration;
using RabbitMQ.Client;
using weatherService.Models.Params;

namespace weatherService.AsyncDataServices
{
    public class MessageBusClient : IMessageBusClient
    {
        private readonly IConfiguration _configuration;
        private readonly IConnection _connection;
        private readonly IModel _channel;

        public MessageBusClient(IConfiguration configuration)
        {
            _configuration = configuration;
            var factory = new ConnectionFactory()
            {
                HostName = _configuration["RabbitMQHost"],
                Port = int.Parse(_configuration["RabbitMQPort"])
            };
            try
            {
                _connection = factory.CreateConnection();
                _channel = _connection.CreateModel();
                _channel.ExchangeDeclare(exchange: "Trigger", type: ExchangeType.Fanout);
                _connection.ConnectionShutdown += RabbitMQ_ConnectionShutdown;
                Console.WriteLine("--> Connection to MessageBus");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"--> Cound not connect to Message Bus: {ex.Message}");
            }
        }

        public void SendRequestToGetAddress(GpsParam param)
        {
            var message = JsonSerializer.Serialize(param);
            if (_connection.IsOpen)
            {
                Console.WriteLine("--> RabbitMQ Connection Open, sending message... ");
                SendMessage(message);
            }
            else
            {
                Console.WriteLine("--> RabbitMQ Connection Closed, not sending");
            }
        }

        private void SendMessage(string message)
        {
            var body = Encoding.UTF8.GetBytes(message);
            _channel.BasicPublish(exchange: "Trigger",
            routingKey: "",
            basicProperties: null,
            body: body);

            Console.WriteLine($"--> We have send {message}");
        }

        public void Dispose()
        {
            Console.WriteLine("MessageBus Disposed");
            if (_channel.IsOpen)
            {
                _channel.Close();
                _connection.Close();
            }
        }
        private void RabbitMQ_ConnectionShutdown(object sender, ShutdownEventArgs e)
        {
            Console.WriteLine("--> RabbitMQ Connection Shutdown");
        }
    }
}
```

可以用以下語法進行新增訊息發佈功能

```b
C:\yo himsk8s:asyncsender
```

依照指數輸入專案名稱及RabbitMQ's exchange name之後，會產生以下兩個檔案

AsyncDataServices/IMessageBusClient.cs

AsyncDataServices/MessageBusClient.cs

* 專案增加RabbitMQ Client的引用

```bash
dotnet add package RabbitMQ.Client
```

* 增加RabbitMQ的設定

  * appsettings.Development.json

    ```json
    {  
      "RabbitMQHost": "localhost",
      "RabbitMQPort": "5672"
    }
    ```

  * appsettings.json

    ```json
    {  
      "RabbitMQHost": "rabbitmq-clusterip-srv",
      "RabbitMQPort": "5672"
    }
    ```

  * 在Staerup.cs ==> public void ConfigureServices 增加以下設定

    並且增加引用處理(QueryService請自行改成專案中的namespace)

    ```c#
    using QueryService.Services.AsyncDataServices;
    namespace QueryService
    {
      ...
        public class Startup
        {
          ...
            public void ConfigureServices(IServiceCollection services)
            {
    		  ...
                services.AddControllers();
                services.AddSingleton<IMessageBusClient, MessageBusClient>();
                ...
            }
        }
    }
    ```

  * 發送訊息

    ```c#
            private readonly IMessageBusClient _messageBusClient;
            public QueryAddrData(IConfiguration configuration, IMessageBusClient messageBusClient)
            {
                Console.WriteLine("QueryAddrData Init");
                _config = configuration;
                _messageBusClient = messageBusClient;
            }
    
    
            public ResultDTO<object> QueryAddrDataToQueue()
            {
    		...
                            _messageBusClient.SendRequestToGetGps(addr);
              ...
            }
    ```

* 建立RabbitMQ訊息訂閱

  可以用以下語法進行新增訊息發佈功能

  ``` bash
  C:\yo himsk8s:asyncsubscriber
  ```

  執行完成新增以下三個檔案

  * AsyncDataService/MessageBusSubscriber.cs 
  
  * EventProcessing/EventProcessor.cs
  
  * EventProcessing/IEventProcessor.cs
  
  * 設定Startup.cs，設定引用來源，
  
    ```c#
    using TransService.Services;
    using TransService.Services.EventProcessing;
    using TransService.Services.AsyncDataService;
    
    namespace TransService
    {
        public class Startup
        {
          ...
            public void ConfigureServices(IServiceCollection services)
            {
    		  ...
                services.AddSingleton<IEventProcessor, EventProcessor>();
                services.AddScoped<ITransAddrToGps, TransAddrToGps>();
                services.AddControllers();
                object p = services.AddHostedService<MessageBusSubscriber>();
            	...
            }
    	...
        }
    }
    ```
  
    專案增加RabbitMQ Client的引用
  
    ``` bash
    dotnet add package RabbitMQ.Client
    ```
  
  * 增加RabbitMQ的設定
  
    * appsettings.Development.json
  
      ```json
      {  
        "RabbitMQHost": "localhost",
        "RabbitMQPort": "5672"
      }
      ```
  
    * appsettings.json
  
      ```json
      {  
        "RabbitMQHost": "rabbitmq-clusterip-srv",
        "RabbitMQPort": "5672"
      }
      ```

測試看看，在查詢及轉送的兩個服務執行以下指令

```bash
dotnet run
```

* EventProcessor增加呼叫TDX進行經緯度轉換服務

  ```c#
  using System;
  using TransService.Services.AsyncDataService;
  using Microsoft.Extensions.Configuration;
  using Newtonsoft.Json;
  using Newtonsoft.Json.Linq;
  using CommonService.Dtos;
  using CommonService.Params;
  using TransService.Params;
  using CommonService.Utilities;
  
  namespace TransService.Services.EventProcessing
  {
      public class EventProcessor : IEventProcessor
      {
          public IConfiguration _config { get; }
  
  
          public EventProcessor(IConfiguration config)
          {
              _config = config;
          }
  
          public void ProcessEvent(string message)
          {
              Console.WriteLine($"--> Event Processing get data: {message}");
  
              AddrParam addrParam = JsonConvert.DeserializeObject<AddrParam>(message);
  
              OpenDataParam param = new OpenDataParam();
              param.Format = "JSON";
              param.Url = _config["TdxBaseApi"].Trim() + _config["TdxAddrToGPS"].Trim() + addrParam.ADDRESS + "?$format=JSON";
              param.Encoding = "utf-8";
              param.Method = "GET";
              param.PostData = "";
  
              ResultDTO<JObject> gpsResult = new ResultDTO<JObject>();
              gpsResult = new OpenDataService().GetOpenData(param);
  
              Console.WriteLine($"--> Event Processing get data: {JsonConvert.SerializeObject(gpsResult)}");
          }
      }
  }
  ```

## Ingress-Nginx

* 安裝Ingress-Nginx

  * 開啟網頁  https://kubernetes.github.io/ingress-nginx/deploy/

  * 移至Docker Desktop的安裝說明區塊

  * 執行以下語法進行Ingress-Nginx安裝

    ``` bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.3/deploy/static/provider/cloud/deploy.yaml
    ```

  * 增加ingress-src.yaml的路由對應設定

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-srv
      annotations:
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/use-regex: 'true'
    spec:
      rules:
        - host: gisservice.com
          http:
            paths:
              - path: /api/qistrans
                pathType: Prefix
                backend:
                  service:
                    name: gisquery-clusterip-srv
                    port:
                      number: 80
              - path: /api/gistransaddr
                pathType: Prefix
                backend:
                  service:
                    name: gisquery-clusterip-srv
                    port:
                      number: 80
    ```

  * 執行路由發行設定

    ```bash
    kubectl apply -f ingress-srv.yaml
    ```

  * 測試路由

    修改Host對應，增加一行『127.0.0.1 gisservice.com』對應

    Mac的Host路徑如下：/private/etc/hosts

    Windows的Host路徑如下：C:\Windows\System32\drivers\etc\hosts



