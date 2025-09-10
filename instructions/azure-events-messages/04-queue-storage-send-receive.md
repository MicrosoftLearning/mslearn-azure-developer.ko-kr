---
lab:
  topic: Azure events and messaging
  title: Azure Queue Storage에서 메시지 보내고 받기
  description: .NET Azure.StorageQueues SDK를 사용하여 Azure Queue Storage에서 메시지를 보내는 방법을 알아봅니다.
---

# Azure Queue Storage에서 메시지 보내고 받기

이 연습에서는 Azure Queue Storage 리소스를 만들고 구성한 다음, **Azure.Storage.Queues** SDK를 사용하여 메시지를 보내고 받는 .NET 앱을 빌드해 보세요. 작업이 완료되면 스토리지 리소스를 프로비전하고, 큐 메시지를 관리하고, 환경을 정리하는 방법을 알아봅니다. 

이 연습에서 수행된 작업:

* Azure Queue Storage 리소스 만들기
* Microsoft Entra 사용자 이름에 역할 할당
* 메시지를 보내고 받기 위한 .NET 콘솔 앱 만들기
* 리소스 정리

이 연습을 완료하는 데 약 **30**분이 걸립니다.

## Azure Queue Storage 리소스 만들기

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
    storAcctName=storactname$RANDOM
    ```

1. 이 연습의 뒷부분에서 스토리지 계정에 할당된 이름이 필요합니다. 다음 명령을 실행하고 출력을 기록해 보세요.

    ```
    echo $storAcctName
    ```

1. 이전에 만든 변수를 사용하여 스토리지 계정을 만들려면 다음 명령을 실행하세요. 이 작업을 완료하는 데 몇 분이 걸립니다.

    ```bash
    az storage account create --resource-group $resourceGroup \
        --name $storAcctName --location $location --sku Standard_LRS
    ```

### Microsoft Entra 사용자 이름에 역할 할당

앱에서 메시지를 보내고 받을 수 있도록 하려면 Microsoft Entra 사용자에게 **Storage 큐 데이터 Contributor** 역할을 할당하세요. 이렇게 하면 사용자 계정에 큐를 만들고 Azure RBAC를 사용하여 메시지를 보내고 받을 수 있는 권한이 부여됩니다. Cloud Shell에서 다음 단계를 수행합니다.

1. 다음 명령을 실행하여 계정에서 **userPrincipalName**을 검색합니다. 이는 역할이 누구에게 할당될 것인지를 나타냅니다.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 다음 명령을 실행하여 스토리지 계정의 리소스 ID를 검색합니다. 리소스 ID는 역할 할당의 범위를 특정 네임스페이스로 설정합니다.

    ```
    resourceID=$(az storage account show --resource-group $resourceGroup \
        --name $storAcctName --query id --output tsv)
    ```

1. 다음 명령을 실행하여 **Storage 큐 데이터 Contributor** 역할을 만들고 할당합니다.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Queue Data Contributor" \
        --scope $resourceID
    ```

## 메시지를 보내고 받기 위한 .NET 콘솔 앱 만들기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. Cloud Shell에서 다음 단계를 수행합니다.

>**팁:** 위쪽 테두리를 끌어서 Cloud Shell의 크기를 조절하면 더 많은 정보와 코드가 표시됩니다. 최소화 및 최대화 단추를 사용하여 Cloud Shell과 기본 포털 인터페이스 사이를 전환할 수도 있습니다.

1. 다음 명령을 실행하여 프로젝트를 포함할 디렉터리를 만들고 해당 프로젝트 디렉터리로 이동합니다.

    ```
    mkdir queuestor
    cd queuestor
    ```

1. .NET 콘솔 애플리케이션을 만들어 보세요.

    ```
    dotnet new console
    ```

1. 다음 명령을 실행하여 프로젝트에 **Azure.Storage.Queues** 및 **Azure.Identity** 패키지를 추가합니다.

    ```
    dotnet add package Azure.Storage.Queues
    dotnet add package Azure.Identity
    ```

### 프로젝트의 시작 코드 추가

1. 애플리케이션을 편집하려면 Cloud Shell에서 다음 명령을 실행합니다.

    ```
    code Program.cs
    ```

1. 기존 콘텐츠를 다음 코드로 바꿉니다. 코드의 주석을 검토하고 **<YOUR-STORAGE-ACCT-NAME>** 을 이전에 기록해 둔 스토리지 계정 이름으로 바꾸세요.

    ```csharp
    using Azure;
    using Azure.Identity;
    using Azure.Storage.Queues;
    using Azure.Storage.Queues.Models;
    using System;
    using System.Threading.Tasks;
    
    // Create a unique name for the queue
    // TODO: Replace the <YOUR-STORAGE-ACCT-NAME> placeholder 
    string queueName = "myqueue-" + Guid.NewGuid().ToString();
    string storageAccountName = "<YOUR-STORAGE-ACCT-NAME>";
    
    // ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE
    
    
    
    // ADD CODE TO SEND AND LIST MESSAGES
    
    
    
    // ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES
    
    
    
    // ADD CODE TO DELETE MESSAGES AND THE QUEUE
    
    
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장합니다.

### 큐 클라이언트를 만들고 큐를 만들기 위한 코드 추가

이제 큐 스토리지 클라이언트를 만들고 큐를 만들기 위한 코드를 추가할 차례입니다.

1. **// ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Instantiate a QueueClient to create and interact with the queue
    QueueClient queueClient = new QueueClient(
        new Uri($"https://{storageAccountName}.queue.core.windows.net/{queueName}"),
        new DefaultAzureCredential(options));
    
    Console.WriteLine($"Creating queue: {queueName}");
    
    // Create the queue
    await queueClient.CreateAsync();
    
    Console.WriteLine("Queue created, press Enter to add messages to the queue...");
    Console.ReadLine();
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, 연습을 계속하세요.

### 큐에서 메시지를 보내고 나열하는 코드 추가

1. **// ADD CODE TO SEND AND LIST MESSAGES** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Send several messages to the queue with the SendMessageAsync method.
    await queueClient.SendMessageAsync("Message 1");
    await queueClient.SendMessageAsync("Message 2");
    
    // Send a message and save the receipt for later use
    SendReceipt receipt = await queueClient.SendMessageAsync("Message 3");
    
    Console.WriteLine("Messages added to the queue. Press Enter to peek at the messages...");
    Console.ReadLine();
    
    // Peeking messages lets you view the messages without removing them from the queue.
    
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to update a message in the queue...");
    Console.ReadLine();
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, 연습을 계속하세요.

### 메시지를 업데이트하고 결과를 나열하는 코드 추가

1. **// ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Update a message with the UpdateMessageAsync method and the saved receipt
    await queueClient.UpdateMessageAsync(receipt.MessageId, receipt.PopReceipt, "Message 3 has been updated");
    
    Console.WriteLine("Message three updated. Press Enter to peek at the messages again...");
    Console.ReadLine();
    
    
    // Peek messages from the queue to compare updated content
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to delete messages from the queue...");
    Console.ReadLine();
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, 연습을 계속하세요.

### 메시지 및 큐를 삭제하는 코드 추가

1. **// ADD CODE TO DELETE MESSAGES AND THE QUEUE** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Delete messages from the queue with the DeleteMessagesAsync method.
    foreach (var message in (await queueClient.ReceiveMessagesAsync(maxMessages: 10)).Value)
    {
        // "Process" the message
        Console.WriteLine($"Deleting message: {message.MessageText}");
    
        // Let the service know we're finished with the message and it can be safely deleted.
        await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    }
    Console.WriteLine("Messages deleted from the queue.");
    Console.WriteLine("\nPress Enter key to delete the queue...");
    Console.ReadLine();
    
    // Delete the queue with the DeleteAsync method.
    Console.WriteLine($"Deleting queue: {queueClient.Name}");
    await queueClient.DeleteAsync();
    
    Console.WriteLine("Done");
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, **Ctrl+Q**를 눌러 편집기를 종료합니다.

## Azure에 로그인하고 앱 실행

1. Cloud Shell 명령줄 창에서 다음 명령을 입력하여 Azure에 로그인합니다.

    ```
    az login
    ```

    **<font color="red">Cloud Shell 세션이 이미 인증되었더라도 Azure에 로그인해야 합니다.</font>**

    > **참고**: 대부분의 시나리오에서는 *az login*을 사용하는 것만으로도 충분합니다. 그러나 여러 테넌트에 구독이 있는 경우 *--tenant* 매개 변수를 사용하여 테넌트 지정해야 할 수 있습니다. 자세한 내용은 [Azure CLI를 사용하여 대화형으로 Azure에 로그인](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)을 참조하세요.

1. 다음 명령을 실행하여 콘솔 앱을 시작합니다. 앱은 실행 도중 여러 번 일시 중지되며, 계속하려면 아무 키나 눌러야 합니다. 이렇게 하면 Azure Portal 메시지를 볼 수 있습니다.

    ```
    dotnet run
    ```

1. Azure Portal에서 만든 Azure Storage 계정으로 이동합니다. 

1. 왼쪽 탐색에서 **> 데이터 스토리지**를 확장하고 **큐**를 선택하세요.

1. 애플리케이션이 만든 큐를 선택하여 보낸 메시지를 보고 애플리케이션이 수행하는 작업을 모니터링할 수 있습니다.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.
1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.

