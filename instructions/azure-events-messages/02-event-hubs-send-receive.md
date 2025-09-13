---
lab:
  topic: Azure events and messaging
  title: Azure Event Hubs에서 이벤트 보내기 및 검색하기
  description: .NET Azure.Messaging.EventHubs SDK를 사용하여 Azure Event Hubs에서 이벤트를 보내고 검색하는 방법을 알아봅니다.
---

# Azure Event Hubs에서 이벤트 보내기 및 검색하기

이 연습에서는 **Azure.Messaging.EventHubs** SDK를 사용하여 Azure Event Hubs 리소스를 만들고, 이벤트를 보내고 받는 .NET 콘솔 앱을 빌드해 보세요. 클라우드 리소스를 프로비전하고, Event Hubs와 상호 작용하고, 작업이 완료된 후 환경을 정리하는 방법을 알아봅니다.

이 연습에서 수행된 작업:

* 리소스 그룹 만들기
* Azure Event Hubs 리소스 만들기
* 이벤트를 보내고 검색하기 위한 .NET 콘솔 앱 만들기
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
    namespaceName=eventhubsns$RANDOM
    ```

### Azure Event Hubs 네임스페이스 및 이벤트 허브 만들기

Azure Event Hubs 네임스페이스는 Azure 내의 이벤트 허브 리소스를 위한 논리적 컨테이너입니다. 이는 하나 이상의 이벤트 허브를 만들 수 있는 고유한 범위 지정 컨테이너를 제공하며, 이 이벤트 허브는 대량의 이벤트 데이터를 수집, 처리, 저장하는 데 사용됩니다. Cloud Shell에서 다음 지침을 수행합니다. 

1. 다음 명령을 실행하여 Event Hubs 네임스페이스를 만듭니다.

    ```
    az eventhubs namespace create --name $namespaceName --resource-group $resourceGroup -l $location
    ```

1. 다음 명령을 실행하여 Event Hubs 네임스페이스에 **myEventHub**라는 이름의 이벤트 허브를 만듭니다. 

    ```
    az eventhubs eventhub create --name myEventHub --resource-group $resourceGroup \
      --namespace-name $namespaceName
    ```

### Microsoft Entra 사용자 이름에 역할 할당

앱에서 메시지를 보내고 받을 수 있도록 하려면, Event Hubs 네임스페이스 수준에서 **Azure Event Hubs 데이터 소유자** 역할에 Microsoft Entra 사용자를 할당하세요. 이렇게 하면 사용자 계정에 Azure RBAC를 사용하여 큐와 항목을 관리하고 액세스할 수 있는 권한이 부여됩니다. Cloud Shell에서 다음 단계를 수행합니다.

1. 다음 명령을 실행하여 계정에서 **userPrincipalName**을 검색합니다. 이는 역할이 누구에게 할당될 것인지를 나타냅니다.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 다음 명령을 실행하여 Event Hubs 네임스페이스의 리소스 ID를 검색합니다. 리소스 ID는 역할 할당의 범위를 특정 네임스페이스로 설정합니다.

    ```
    resourceID=$(az eventhubs namespace show --resource-group $resourceGroup \
        --name $namespaceName --query id --output tsv)
    ```
1. 다음 명령을 실행하여 이벤트를 보내고 검색할 수 있는 권한을 부여하는 **Azure Event Hubs 데이터 소유자** 역할을 만들고 할당합니다.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Event Hubs Data Owner" \
        --scope $resourceID
    ```

## .NET 콘솔 애플리케이션을 사용하여 이벤트 보내기 및 검색하기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. Cloud Shell에서 다음 단계를 수행합니다.

>**팁:** 위쪽 테두리를 끌어서 Cloud Shell의 크기를 조절하면 더 많은 정보와 코드가 표시됩니다. 최소화 및 최대화 단추를 사용하여 Cloud Shell과 기본 포털 인터페이스 사이를 전환할 수도 있습니다.

1. 다음 명령을 실행하여 프로젝트를 포함할 디렉터리를 만들고 해당 프로젝트 디렉터리로 이동합니다.

    ```
    mkdir eventhubs
    cd eventhubs
    ```

1. .NET 콘솔 애플리케이션을 만들어 보세요.

    ```
    dotnet new console
    ```

1. 다음 명령을 실행하여 프로젝트에 **Azure.Messaging.EventHubs** 및 **Azure.Identity** 패키지를 추가합니다.

    ```
    dotnet add package Azure.Messaging.EventHubs
    dotnet add package Azure.Identity
    ```

이제 Cloud Shell의 편집기를 사용하여 **Program.cs** 파일의 템플릿 코드를 바꿔야 합니다.

### 프로젝트의 시작 코드 추가

1. 애플리케이션을 편집하려면 Cloud Shell에서 다음 명령을 실행합니다.

    ```
    code Program.cs
    ```

1. 기존 콘텐츠를 다음 코드로 바꿉니다. 코드의 주석을 검토하고 **YOUR_EVENT_HUB_NAMESPACE**를 이벤트 허브 네임스페이스로 바꿉니다.

    ```csharp
    using Azure.Messaging.EventHubs;
    using Azure.Messaging.EventHubs.Producer;
    using Azure.Messaging.EventHubs.Consumer;
    using Azure.Identity;
    using System.Text;
    
    // TO-DO: Replace YOUR_EVENT_HUB_NAMESPACE with your actual Event Hub namespace
    string namespaceURL = "YOUR_EVENT_HUB_NAMESPACE.servicebus.windows.net";
    string eventHubName = "myEventHub"; 
    
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Number of events to be sent to the event hub
    int numOfEvents = 3;
    
    // CREATE A PRODUCER CLIENT AND SEND EVENTS
    
    
    
    // CREATE A CONSUMER CLIENT AND RECEIVE EVENTS
    
    
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장합니다.

### 애플리케이션을 완성하기 위한 코드 추가

이 섹션에서는 이벤트를 보내고 받는 생산자 및 소비자 클라이언트를 만드는 코드를 추가합니다.

1. **// CREATE A PRODUCER CLIENT AND SEND EVENTS** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드의 주석을 반드시 검토하세요.

    ```csharp
    // Create a producer client to send events to the event hub
    EventHubProducerClient producerClient = new EventHubProducerClient(
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    // Create a batch of events 
    using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();
    
    
    // Adding a random number to the event body and sending the events. 
    var random = new Random();
    for (int i = 1; i <= numOfEvents; i++)
    {
        int randomNumber = random.Next(1, 101); // 1 to 100 inclusive
        string eventBody = $"Event {randomNumber}";
        if (!eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes(eventBody))))
        {
            // if it is too large for the batch
            throw new Exception($"Event {i} is too large for the batch and cannot be sent.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of events to the event hub
        await producerClient.SendAsync(eventBatch);
    
        Console.WriteLine($"A batch of {numOfEvents} events has been published.");
        Console.WriteLine("Press Enter to retrieve and print the events...");
        Console.ReadLine();
    }
    finally
    {
        await producerClient.DisposeAsync();
    }
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장합니다.

1. **// CREATE A CONSUMER CLIENT AND RETRIEVE EVENTS** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드의 주석을 반드시 검토하세요.

    ```csharp
    // Create an EventHubConsumerClient
    await using var consumerClient = new EventHubConsumerClient(
        EventHubConsumerClient.DefaultConsumerGroupName,
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    Console.Clear();
    Console.WriteLine("Retrieving all events from the hub...");
    
    // Get total number of events in the hub by summing (last - first + 1) for all partitions
    // This count is used to determine when to stop reading events
    long totalEventCount = 0;
    string[] partitionIds = await consumerClient.GetPartitionIdsAsync();
    foreach (var partitionId in partitionIds)
    {
        PartitionProperties properties = await consumerClient.GetPartitionPropertiesAsync(partitionId);
        if (!properties.IsEmpty && properties.LastEnqueuedSequenceNumber >= properties.BeginningSequenceNumber)
        {
            totalEventCount += (properties.LastEnqueuedSequenceNumber - properties.BeginningSequenceNumber + 1);
        }
    }
    
    // Start retrieving events from the event hub and print to the console
    int retrievedCount = 0;
    await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsAsync(startReadingAtEarliestEvent: true))
    {
        if (partitionEvent.Data != null)
        {
            string body = Encoding.UTF8.GetString(partitionEvent.Data.Body.ToArray());
            Console.WriteLine($"Retrieved event: {body}");
            retrievedCount++;
            if (retrievedCount >= totalEventCount)
            {
                Console.WriteLine("Done retrieving events. Press Enter to exit...");
                Console.ReadLine();
                return;
            }
        }
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

1. 다음 명령을 실행하여 애플리케이션을 시작합니다.

    ```
    dotnet run
    ```

    몇 초 후에 다음 예와 비슷한 출력이 표시됩니다.
    
    ```
    A batch of 3 events has been published.
    Press Enter to retrieve and print the events...
    
    Retrieving all events from the hub...
    Retrieved event: Event 4
    Retrieved event: Event 96
    Retrieved event: Event 74
    Done retrieving events. Press Enter to exit...
    ```

애플리케이션은 허브로 항상 세 개의 이벤트를 보내지만, 허브에 있는 모든 이벤트를 검색합니다. 애플리케이션을 여러 번 실행하면, 검색되는 이벤트 수도 늘어납니다. 이벤트 생성에 사용되는 난수는 다양한 이벤트를 식별하는 데 도움이 됩니다.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.
1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다. 
