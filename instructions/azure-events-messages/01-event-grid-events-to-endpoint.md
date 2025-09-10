---
lab:
  topic: Azure events and messaging
  title: Azure Event Grid로 이벤트를 사용자 지정 엔드포인트로 라우팅
  description: Azure Event Grid를 사용하여 이벤트를 사용자 지정 엔드포인트로 라우팅하는 방법을 알아봅니다.
---

# Azure Event Grid로 이벤트를 사용자 지정 엔드포인트로 라우팅

이 연습에서는 Azure Event Grid 항목과 웹앱 엔드포인트를 생성한 후, 사용자 지정 이벤트를 Event Grid 항목으로 전송하는 .NET 콘솔 애플리케이션을 작성해 보세요. 이벤트 구독을 구성하고, Event Grid를 사용하여 인증하고, 웹앱에서 이벤트를 확인하여 이벤트가 엔드포인트로 성공적으로 라우팅되는지 확인하는 방법을 알아봅니다.

이 연습에서 수행된 작업:

* Azure Event Grid 리소스 만들기
* Event Grid 리소스 공급자 활성화
* Event Grid에서 항목 만들기
* 메시지 엔드포인트 만들기
* 항목 구독
* .NET 콘솔 앱을 사용하여 이벤트 보내기
* 리소스 정리

이 연습을 완료하는 데 약 **30**분이 걸립니다.

## Azure Event Grid 리소스 만들기

이 연습 섹션에서는 Azure CLI를 사용하여 Azure에서 필요한 리소스를 만듭니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.

1. 페이지 상단의 검색 창 오른쪽에 있는 **[\>_]** 단추를 사용하여 Azure Portal에서 새 Cloud Shell을 만들고 ***Bash*** 환경을 선택합니다. Cloud Shell은 다음과 같이 Azure Portal 아래쪽 창에 명령줄 인터페이스를 제공합니다. 파일을 보관할 스토리지 계정을 선택하라는 프롬프트가 표시되면 **스토리지 계정 필요 없음**, 구독을 차례로 선택한 다음, **적용**을 선택합니다.

    > **참고**: 이전에 *PowerShell* 환경을 사용하는 Cloud Shell을 만든 경우 ***Bash***로 전환합니다.

1. Cloud Shell 도구 모음의 **설정** 메뉴에서 **클래식 버전으로 이동**을 선택합니다(코드 편집기를 사용하는 데 필요).

1. 이 연습에 필요한 리소스에 대한 리소스 그룹을 만듭니다. 사용하려는 리소스 그룹이 이미 있는 경우, 다음 단계를 진행하세요. **myResourceGroup**을 리소스 그룹에 사용하려는 이름으로 바꾸세요. 필요한 경우 **eastus**를 근처 지역으로 바꿀 수 있습니다.

    ```bash
    az group create --name myResourceGroup --location eastus
    ```

1. 대부분의 명령은 고유한 이름이 필요하며 동일한 매개 변수를 사용합니다. 일부 변수를 만들어 두면 리소스를 만드는 명령에 필요한 변경 내용을 줄일 수 있습니다. 다음 명령을 실행하여 필요한 변수를 만들 수 있습니다. **myResourceGroup**을 이 연습에 사용하는 이름으로 바꾸세요. 이전 단계에서 위치를 변경한 경우, **location** 변수에서도 동일한 변경을 적용해야 합니다.

    ```bash
    let rNum=$RANDOM
    resourceGroup=myResourceGroup
    location=eastus
    topicName="mytopic-evgtopic-${rNum}"
    siteName="evgsite-${rNum}"
    siteURL="https://${siteName}.azurewebsites.net"
    ```

### Event Grid 리소스 공급자 활성화

Azure 리소스 공급자는 Azure에서 특정 유형의 리소스를 정의하고 관리하는 서비스입니다. Azure가 리소스를 배포하거나 관리할 때 백그라운드에서 사용하는 서비스입니다. **az provider register** 명령을 사용하여 Event Grid 리소스 공급자를 등록합니다. 

```bash
az provider register --namespace Microsoft.EventGrid
```

등록이 완료되려면 몇 분 정도 걸릴 수 있습니다. 다음 명령을 사용하여 상태를 확인할 수 있습니다.

```bash
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```

> **참고:** 이 단계는 Event Grid를 사용한 적 없는 구독에서만 필요합니다.

### Event Grid에서 항목 만들기

**az eventgrid topic create** 명령을 사용하여 항목을 만듭니다. 이름은 DNS 항목의 일부이므로 고유해야 합니다.  

```bash
az eventgrid topic create --name $topicName \
    --location $location \
    --resource-group $resourceGroup
```

### 메시지 엔드포인트 만들기

사용자 지정 토픽을 구독하기 전에 이벤트 메시지에 대한 엔드포인트를 만들어야 합니다. 일반적으로 엔드포인트는 이벤트 데이터를 기반으로 작업을 수행합니다. 다음 스크립트는 이벤트 메시지를 표시하는 미리 작성된 웹앱을 사용합니다. 배포된 솔루션은 App Service 계획, App Service 웹앱 및 GitHub의 소스 코드를 포함합니다.

1. 다음 명령을 실행하여 메시지 엔드포인트를 만듭니다. **echo** 명령은 엔드포인트의 사이트 URL을 표시합니다.

    ```bash
    az deployment group create \
        --resource-group $resourceGroup \
        --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/main/azuredeploy.json" \
        --parameters siteName=$siteName hostingPlanName=viewerhost
    
    echo "Your web app URL: ${siteURL}"
    ```

    > **참고:** 이 명령을 완료하는 데 몇 분 정도 걸릴 수 있습니다.

1. 브라우저에서 새 탭을 열고 이전 스크립트의 끝에서 생성된 URL로 이동하여 웹앱이 실행 중인지 확인하세요. 현재 표시된 메시지가 없는 사이트가 표시되어야 합니다.

    > **팁:** 브라우저를 실행 상태로 두면 업데이트를 표시하는 데 사용됩니다.

### 항목 구독

Event Grid 토픽을 구독하여 Event Grid에 추적하려는 이벤트와 이벤트를 보낼 위치를 알립니다. 

1. **az eventgrid event-subscription create** 명령을 사용하여 항목을 구독합니다. 다음 스크립트는 계정에서 구독 ID를 검색하여 이벤트 구독을 만드는 데 사용합니다.

    ```bash
    endpoint="${siteURL}/api/updates"
    topicId=$(az eventgrid topic show --resource-group $resourceGroup \
        --name $topicName --query "id" --output tsv)
    
    az eventgrid event-subscription create \
        --source-resource-id $topicId \
        --name TopicSubscription \
        --endpoint $endpoint
    ```

1. 웹앱을 다시 확인하고, 구독 유효성 검사 이벤트를 보냈음을 확인합니다. 눈 모양 아이콘을 선택하여 이벤트 데이터를 확장합니다. Event Grid는 유효성 검사 이벤트를 보내므로 엔드포인트는 이벤트 데이터를 수신하려는 것을 확인할 수 있습니다. 웹앱은 구독의 유효성을 검사하는 코드를 포함합니다.

## .NET 콘솔 애플리케이션을 사용하여 이벤트 보내기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. Cloud Shell에서 다음 단계를 수행합니다.

>**팁:** 위쪽 테두리를 끌어서 Cloud Shell의 크기를 조절하면 더 많은 정보와 코드가 표시됩니다. 최소화 및 최대화 단추를 사용하여 Cloud Shell과 기본 포털 인터페이스 사이를 전환할 수도 있습니다.

1. 다음 명령을 실행하여 프로젝트를 포함할 디렉터리를 만들고 해당 프로젝트 디렉터리로 이동합니다.

    ```bash
    mkdir eventgrid
    cd eventgrid
    ```

1. .NET 콘솔 애플리케이션을 만들어 보세요.

    ```bash
    dotnet new console
    ```

1. 다음 명령을 실행하여 프로젝트에 **Azure.Messaging.EventGrid** 및 **dotenv.net** 패키지를 추가합니다.

    ```bash
    dotnet add package Azure.Messaging.EventGrid
    dotnet add package dotenv.net
    ```

### 콘솔 애플리케이션 구성

이 섹션에서는 해당 비밀을 **.env** 파일에 추가하여 보관할 수 있도록 항목 엔드포인트 및 액세스 키를 검색합니다.

1. 다음 명령을 실행하여 앞서 만든 항목의 URL과 액세스 키를 검색합니다. 이러한 값을 반드시 기록해 두어야 합니다.

    ```bash
    az eventgrid topic show --name $topicName -g $resourceGroup --query "endpoint" --output tsv
    az eventgrid topic key list --name $topicName -g $resourceGroup --query "key1" --output tsv
    ```

1. 다음 명령을 실행하여 비밀을 보관할 **.env** 파일을 만든 다음, 코드 편집기에서 엽니다.

    ```bash
    touch .env
    code .env
    ```

1. **.env** 파일에 다음 코드를 추가합니다. **YOUR_TOPIC_ENDPOINT** 및 **YOUR_TOPIC_ACCESS_KEY**를 이전에 기록한 값으로 바꾸세요.

    ```
    TOPIC_ENDPOINT="YOUR_TOPIC_ENDPOINT"
    TOPIC_ACCESS_KEY="YOUR_TOPIC_ACCESS_KEY"
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, **Ctrl+Q**를 눌러 편집기를 종료합니다.

이제 Cloud Shell의 편집기를 사용하여 **Program.cs** 파일의 템플릿 코드를 바꿔야 합니다.

### 프로젝트용 코드 추가

1. 애플리케이션을 편집하려면 Cloud Shell에서 다음 명령을 실행합니다.

    ```bash
    code Program.cs
    ```

1. 기존 코드를 다음 코드로 바꿉니다. 코드의 주석을 반드시 검토하세요.

    ```csharp
    using dotenv.net; 
    using Azure.Messaging.EventGrid; 
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    
    // Start the asynchronous process to send an Event Grid event
    ProcessAsync().GetAwaiter().GetResult();
    
    async Task ProcessAsync()
    {
        // Retrieve Event Grid topic endpoint and access key from environment variables
        var topicEndpoint = envVars["TOPIC_ENDPOINT"];
        var topicKey = envVars["TOPIC_ACCESS_KEY"];
        
        // Check if the required environment variables are set
        if (string.IsNullOrEmpty(topicEndpoint) || string.IsNullOrEmpty(topicKey))
        {
            Console.WriteLine("Please set TOPIC_ENDPOINT and TOPIC_ACCESS_KEY in your .env file.");
            return;
        }
    
        // Create an EventGridPublisherClient to send events to the specified topic
        EventGridPublisherClient client = new EventGridPublisherClient
            (new Uri(topicEndpoint),
            new Azure.AzureKeyCredential(topicKey));
    
        // Create a new EventGridEvent with sample data
        var eventGridEvent = new EventGridEvent(
            subject: "ExampleSubject",
            eventType: "ExampleEventType",
            dataVersion: "1.0",
            data: new { Message = "Hello, Event Grid!" }
        );
    
        // Send the event to Azure Event Grid
        await client.SendEventAsync(eventGridEvent);
        Console.WriteLine("Event sent successfully.");
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

1. Cloud Shell에서 다음 명령을 실행하여 콘솔 애플리케이션을 시작합니다. **이벤트가 성공적으로 전송되었습니다.** 라는 메시지가 표시됩니다. 메시지 전송 시점

    ```bash
    dotnet run
    ```

1. 웹앱을 확인하여 방금 전송한 이벤트를 봅니다. 눈 모양 아이콘을 선택하여 이벤트 데이터를 확장합니다.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.
1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.