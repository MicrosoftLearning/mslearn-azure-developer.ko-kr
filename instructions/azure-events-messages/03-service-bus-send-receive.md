---
lab:
  topic: Azure events and messaging
  title: Azure Service Bus에서 메시지 보내고 받기
  description: .NET Azure.Messaging.ServiceBus SDK를 사용하여 Azure Service Bus에서 메시지를 보내는 방법을 알아봅니다.
---

# Azure Service Bus에서 메시지 보내고 받기

이 연습에서는 Azure Service Bus 리소스를 만들고 구성한 다음, **Azure.Messaging.ServiceBus** SDK를 사용하여 메시지를 보내고 받는 .NET 앱을 빌드해 보세요. Service Bus 네임스페이스와 큐를 프로비전하고, 권한을 할당하고, 프로그래밍 방식으로 메시지와 상호 작용하는 방법을 알아봅니다. 

이 연습에서 수행된 작업:

* Azure Service Bus 리소스 만들기
* Microsoft Entra 사용자 이름에 역할 할당
* 메시지를 보내고 받기 위한 .NET 콘솔 앱 만들기
* 리소스 정리

이 연습을 완료하는 데 약 **30**분이 걸립니다.

## Azure Event Hubs 리소스 만들기

이 연습 섹션에서는 Azure CLI를 사용하여 Azure에서 필요한 리소스를 만듭니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.

1. 페이지 상단의 검색 창 오른쪽에 있는 **[\>_]** 단추를 사용하여 Azure Portal에서 새 Cloud Shell을 만들고 ***Bash*** 환경을 선택합니다. Cloud Shell은 다음과 같이 Azure Portal 아래쪽 창에 명령줄 인터페이스를 제공합니다. 파일을 보관할 스토리지 계정을 선택하라는 프롬프트가 표시되면 **스토리지 계정 필요 없음**, 구독을 차례로 선택한 다음, **적용**을 선택합니다.

    > **참고**: 이전에 *PowerShell* 환경을 사용하는 Cloud Shell을 만든 경우 ***Bash***로 전환합니다.

1. Cloud Shell 도구 모음의 **설정** 메뉴에서 **클래식 버전으로 이동**을 선택합니다(코드 편집기를 사용하는 데 필요).

1. 이 연습에 필요한 리소스에 대한 리소스 그룹을 만듭니다. 사용하려는 리소스 그룹이 이미 있는 경우, 다음 단계를 진행하세요. **myResourceGroup**을 리소스 그룹에 사용하려는 이름으로 바꾸세요. 필요한 경우 **eastus**를 근처 지역으로 바꿀 수 있습니다.

    ```
    az group create --name myResourceGroup --location eastus
    ```

1. 대부분의 명령은 고유한 이름이 필요하며 동일한 매개 변수를 사용합니다. 일부 변수를 만들어 두면 리소스를 만드는 명령에 필요한 변경 내용을 줄일 수 있습니다. 다음 명령을 실행하여 필요한 변수를 만들 수 있습니다. **myResourceGroup**을 이 연습에 사용하는 이름으로 바꾸세요. 이전 단계에서 위치를 변경한 경우, **location** 변수에서도 동일한 변경을 적용해야 합니다.

    ```
    resourceGroup=myResourceGroup
    location=eastus
    namespaceName=svcbusns$RANDOM
    ```

1. 이 연습의 뒷부분에서 네임스페이스에 할당된 이름이 필요합니다. 다음 명령을 실행하고 출력을 기록해 보세요.

    ```
    echo $namespaceName
    ```

### Azure Service Bus 네임스페이스 및 큐 만들기

1. Service Bus 메시징 네임스페이스를 만듭니다. 아래 명령은 이전에 만든 변수를 사용하여 네임스페이스를 만듭니다. 이 작업을 완료하는 데 몇 분이 걸립니다.

    ```bash
    az servicebus namespace create \
        --resource-group $resourceGroup \
        --name $namespaceName \
        --location $location
    ```

1. 이제 네임스페이스가 생성되었으므로 메시지를 보관할 큐를 만들어야 합니다. 다음 명령을 실행하여 **myqueue**라는 이름의 큐를 만듭니다.

    ```bash
    az servicebus queue create --resource-group $resourceGroup \
        --namespace-name $namespaceName \
        --name myqueue
    ```

### Microsoft Entra 사용자 이름에 역할 할당

앱에서 메시지를 보내고 받을 수 있도록 하려면, Service Bus 네임스페이스 수준에서 **Azure Service Bus 데이터 소유자** 역할에 Microsoft Entra 사용자를 할당하세요. 이렇게 하면 사용자 계정에 Azure RBAC를 사용하여 큐와 항목을 관리하고 액세스할 수 있는 권한이 부여됩니다. Cloud Shell에서 다음 단계를 수행합니다.

1. 다음 명령을 실행하여 계정에서 **userPrincipalName**을 검색합니다. 이는 역할이 누구에게 할당될 것인지를 나타냅니다.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 다음 명령을 실행하여 Service Bus 네임스페이스의 리소스 ID를 검색합니다. 리소스 ID는 역할 할당의 범위를 특정 네임스페이스로 설정합니다.

    ```
    resourceID=$(az servicebus namespace show --name $namespaceName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. 다음 명령을 실행하여 **Azure Service Bus 데이터 소유자** 역할을 만들고 할당합니다.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Service Bus Data Owner" \
        --scope $resourceID
    ```

## 메시지를 보내고 받기 위한 .NET 콘솔 앱 만들기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. Cloud Shell에서 다음 단계를 수행합니다.

>**팁:** 위쪽 테두리를 끌어서 Cloud Shell의 크기를 조절하면 더 많은 정보와 코드가 표시됩니다. 최소화 및 최대화 단추를 사용하여 Cloud Shell과 기본 포털 인터페이스 사이를 전환할 수도 있습니다.

1. 다음 명령을 실행하여 프로젝트를 포함할 디렉터리를 만들고 해당 프로젝트 디렉터리로 이동합니다.

    ```
    mkdir svcbus
    cd svcbus
    ```

1. .NET 콘솔 애플리케이션을 만들어 보세요.

    ```
    dotnet new console
    ```

1. 다음 명령을 실행하여 프로젝트에 **Azure.Messaging.ServiceBus** 및 **Azure.Identity** 패키지를 추가합니다.

    ```
    dotnet add package Azure.Messaging.ServiceBus
    dotnet add package Azure.Identity
    ```

### 프로젝트의 시작 코드 추가

1. 애플리케이션을 편집하려면 Cloud Shell에서 다음 명령을 실행합니다.

    ```
    code Program.cs
    ```

1. 기존 콘텐츠를 다음 코드로 바꿉니다. 코드의 주석을 검토하고 **<YOUR-NAMESPACE>** 를 이전에 기록한 Service Bus 네임스페이스로 바꾸세요.

    ```csharp
    using Azure.Messaging.ServiceBus;
    using Azure.Identity;
    using System.Timers;
    
    
    // TODO: Replace <YOUR-NAMESPACE> with your Service Bus namespace
    string svcbusNameSpace = "<YOUR-NAMESPACE>.servicebus.windows.net";
    string queueName = "myQueue";
    
    
    // ADD CODE TO CREATE A SERVICE BUS CLIENT
    
    
    
    // ADD CODE TO SEND MESSAGES TO THE QUEUE
    
    
    
    // ADD CODE TO PROCESS MESSAGES FROM THE QUEUE
    
    
    
    // Dispose client after use
    await client.DisposeAsync();
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장합니다.

### 메시지를 큐에 보내는 코드 추가

이제 Service Bus 클라이언트를 만들고 일괄 처리된 메시지를 큐로 보내는 코드를 추가할 차례입니다.

1. **// ADD CODE TO CREATE A SERVICE BUS CLIENT** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a Service Bus client using the namespace and DefaultAzureCredential
    // The DefaultAzureCredential will use the Azure CLI credentials, so ensure you are logged in
    ServiceBusClient client = new(svcbusNameSpace, new DefaultAzureCredential(options));
    ```

1. **// ADD CODE TO SEND MESSAGES TO THE QUEUE** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Create a sender for the specified queue
    ServiceBusSender sender = client.CreateSender(queueName);
    
    // create a batch 
    using ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync();
    
    // number of messages to be sent to the queue
    const int numOfMessages = 3;
    
    for (int i = 1; i <= numOfMessages; i++)
    {
        // try adding a message to the batch
        if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
        {
            // if it is too large for the batch
            throw new Exception($"The message {i} is too large to fit in the batch.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of messages to the Service Bus queue
        await sender.SendMessagesAsync(messageBatch);
        Console.WriteLine($"A batch of {numOfMessages} messages has been published to the queue.");
    }
    finally
    {
        // Calling DisposeAsync on client types is required to ensure that network
        // resources and other unmanaged objects are properly cleaned up.
        await sender.DisposeAsync();
    }
    
    Console.WriteLine("Press any key to continue");
    Console.ReadKey();
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, 연습을 계속하세요.

### 큐에 있는 메시지를 처리하는 코드 추가

1. **// ADD CODE TO PROCESS MESSAGES FROM THE QUEUE** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Create a processor that we can use to process the messages in the queue
    ServiceBusProcessor processor = client.CreateProcessor(queueName, new ServiceBusProcessorOptions());
    
    // Idle timeout in milliseconds, the idle timer will stop the processor if there are no more 
    // messages in the queue to process
    const int idleTimeoutMs = 3000;
    System.Timers.Timer idleTimer = new(idleTimeoutMs);
    idleTimer.Elapsed += async (s, e) =>
    {
        Console.WriteLine($"No messages received for {idleTimeoutMs / 1000} seconds. Stopping processor...");
        await processor.StopProcessingAsync();
    };
    
    try
    {
        // add handler to process messages
        processor.ProcessMessageAsync += MessageHandler;
    
        // add handler to process any errors
        processor.ProcessErrorAsync += ErrorHandler;
    
        // start processing 
        idleTimer.Start();
        await processor.StartProcessingAsync();
    
        Console.WriteLine($"Processor started. Will stop after {idleTimeoutMs / 1000} seconds of inactivity.");
        // Wait for the processor to stop
        while (processor.IsProcessing)
        {
            await Task.Delay(500);
        }
        idleTimer.Stop();
        Console.WriteLine("Stopped receiving messages");
    }
    finally
    {
        // Dispose processor after use
        await processor.DisposeAsync();
    }
    
    // handle received messages
    async Task MessageHandler(ProcessMessageEventArgs args)
    {
        string body = args.Message.Body.ToString();
        Console.WriteLine($"Received: {body}");
    
        // Reset the idle timer on each message
        idleTimer.Stop();
        idleTimer.Start();
    
        // complete the message. message is deleted from the queue. 
        await args.CompleteMessageAsync(args.Message);
    }
    
    // handle any errors when receiving messages
    Task ErrorHandler(ProcessErrorEventArgs args)
    {
        Console.WriteLine(args.Exception.ToString());
        return Task.CompletedTask;
    }
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, **Ctrl+Q**를 눌러 편집기를 종료합니다.

## Azure에 로그인하고 앱 실행

1. Cloud Shell 명령줄 창에서 다음 명령을 입력하여 Azure에 로그인합니다.

    ```
    az login
    ```

    **<font color="red">Cloud Shell 세션이 이미 인증되었더라도 Azure에 로그인해야 합니다.</font>**

    > **참고**: 대부분의 시나리오에서는 *az login*을 사용하는 것만으로도 충분합니다. 그러나 여러 테넌트에 구독이 있는 경우 *--tenant* 매개 변수를 사용하여 테넌트 지정해야 할 수 있습니다. 자세한 내용은 [Azure CLI를 사용하여 대화형으로 Azure에 로그인](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)을 참조하세요.

1. 다음 명령을 실행하여 콘솔 앱을 시작합니다. 앱은 다양한 단계를 일시 정지하고 계속하려면 키를 누르라는 프롬프트를 표시합니다. 이렇게 하면 Azure Portal 메시지를 볼 수 있습니다.

    ```
    dotnet run
    ```

    

1. Azure Portal에서 만든 Service Bus 네임스페이스로 이동합니다. 

1. **개요** 창 아래에서 **myqueue**를 선택하세요.

1. 왼쪽 탐색 창에서 **Service Bus Explorer**를 선택하세요.

1. **시작에서 피킹**을 선택하면 몇 초 후에 세 개의 메시지가 나타납니다.

1. Cloud Shell에서 아무 키나 눌러 계속하면 애플리케이션이 세 개의 메시지를 처리합니다. 
 
1. 애플리케이션이 메시지 처리를 완료한 후 포털로 돌아갑니다. 다시 **시작에서 피킹**을 선택하면 큐에 메시지가 없는지 확인합니다.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 범위를 벗어난 기존 리소스는 

