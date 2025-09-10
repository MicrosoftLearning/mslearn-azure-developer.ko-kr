---
lab:
  topic: Azure Cosmos DB
  title: .NET을 사용하여 Azure Cosmos DB for NoSQL의 리소스 만들기
  description: Microsoft .NET SDK v3을 사용하여 Azure Cosmos DB에서 데이터베이스 및 컨테이너 리소스를 만드는 방법을 알아봅니다.
---

# .NET을 사용하여 Azure Cosmos DB for NoSQL의 리소스 만들기

이 연습에서는 Azure Cosmos DB 계정을 만들고 Microsoft Azure Cosmos DB SDK를 사용하여 데이터베이스, 컨테이너, 샘플 항목을 만드는 .NET 콘솔 애플리케이션을 빌드해 보세요. Azure Portal에서 인증을 구성하고, 데이터베이스 작업을 프로그래밍 방식으로 수행하고, 결과를 확인하는 방법을 알아봅니다.

이 연습에서 수행된 작업:

* Azure Cosmos DB 계정 만들기
* 데이터베이스, 컨테이너, 항목을 생성하는 콘솔 앱 만들기
* 콘솔 앱 실행 및 결과 확인

이 연습을 완료하는 데 약 **30**분이 걸립니다.

## Azure Cosmos DB 계정 만들기

이 연습 섹션에서는 리소스 그룹 및 Azure Cosmos DB 계정을 만듭니다. 또한 계정에 대한 엔드포인트 및 액세스 키를 기록합니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.

1. 페이지 상단의 검색 창 오른쪽에 있는 **[\>_]** 단추를 사용하여 Azure Portal에서 새 Cloud Shell을 만들고 ***Bash*** 환경을 선택합니다. Cloud Shell은 다음과 같이 Azure Portal 아래쪽 창에 명령줄 인터페이스를 제공합니다. 파일을 보관할 스토리지 계정을 선택하라는 프롬프트가 표시되면 **스토리지 계정 필요 없음**, 구독을 차례로 선택한 다음, **적용**을 선택합니다.

    > **참고**: 이전에 *PowerShell* 환경을 사용하는 Cloud Shell을 만든 경우 ***Bash***로 전환합니다.

1. Cloud Shell 도구 모음의 **설정** 메뉴에서 **클래식 버전으로 이동**을 선택합니다(코드 편집기를 사용하는 데 필요).

1. 이 연습에 필요한 리소스에 대한 리소스 그룹을 만듭니다. 사용하려는 리소스 그룹이 이미 있는 경우, 다음 단계를 진행하세요. **myResourceGroup**을 리소스 그룹에 사용하려는 이름으로 바꾸세요. 필요한 경우 **eastus**를 근처 지역으로 바꿀 수 있습니다.

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. 대부분의 명령은 고유한 이름이 필요하며 동일한 매개 변수를 사용합니다. 일부 변수를 만들어 두면 리소스를 만드는 명령에 필요한 변경 내용을 줄일 수 있습니다. 다음 명령을 실행하여 필요한 변수를 만들 수 있습니다. **myResourceGroup**을 이 연습에 사용하는 이름으로 바꾸세요.

    ```
    resourceGroup=myResourceGroup
    accountName=cosmosexercise$RANDOM
    ```

1. 다음 명령을 실행하여 Azure Cosmos DB 계정을 만드세요. 각 계정 이름은 고유해야 합니다. 

    ```
    az cosmosdb create --name $accountName \
        --resource-group $resourceGroup
    ```

1.  다음 명령을 실행하여 Azure Cosmos DB 계정에 대한 **documentEndpoint**를 검색합니다. 명령 결과에서 엔드포인트를 기록합니다. 연습의 뒷부분에서 사용됩니다.

    ```
    az cosmosdb show --name $accountName \
        --resource-group $resourceGroup \
        --query "documentEndpoint" --output tsv
    ```

1. 다음 명령을 사용하여 계정의 기본 키를 검색합니다. 명령 결과에서 기본 키를 기록합니다. 연습의 뒷부분에서 사용됩니다.

    ```
    az cosmosdb keys list --name $accountName \
        --resource-group $resourceGroup \
        --query "primaryMasterKey" --output tsv
    ```

## .NET 콘솔 애플리케이션을 사용하여 데이터 리소스 및 항목 만들기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. Cloud Shell에서 다음 단계를 수행합니다.

>**팁:** 위쪽 테두리를 끌어서 Cloud Shell의 크기를 조절하면 더 많은 정보와 코드가 표시됩니다. 최소화 및 최대화 단추를 사용하여 Cloud Shell과 기본 포털 인터페이스 사이를 전환할 수도 있습니다.

1. 프로젝트에 대한 폴더를 만들고 폴더로 변경합니다.

    ```bash
    mkdir cosmosdb
    cd cosmosdb
    ```

1. .NET 콘솔 앱을 만듭니다.

    ```bash
    dotnet new console
    ```

### 콘솔 애플리케이션 구성

1. 다음 명령을 실행하여 프로젝트에 **Microsoft.Azure.Cosmos**, **Newtonsoft.Json**, **dotenv.net** 패키지를 추가합니다.

    ```bash
    dotnet add package Microsoft.Azure.Cosmos --version 3.*
    dotnet add package Newtonsoft.Json --version 13.*
    dotnet add package dotenv.net
    ```

1. 다음 명령을 실행하여 비밀을 보관할 **.env** 파일을 만든 다음, 코드 편집기에서 엽니다.

    ```bash
    touch .env
    code .env
    ```

1. **.env** 파일에 다음 코드를 추가합니다. **YOUR_DOCUMENT_ENDPOINT** 및 **YOUR_ACCOUNT_KEY**를 이전에 기록한 값으로 바꾸세요.

    ```
    DOCUMENT_ENDPOINT="YOUR_DOCUMENT_ENDPOINT"
    ACCOUNT_KEY="YOUR_ACCOUNT_KEY"
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, **Ctrl+Q**를 눌러 편집기를 종료합니다.

이제 Cloud Shell의 편집기를 사용하여 **Program.cs** 파일의 템플릿 코드를 바꿔야 합니다.

### 프로젝트의 시작 코드 추가

1. 애플리케이션을 편집하려면 Cloud Shell에서 다음 명령을 실행합니다.

    ```bash
    code Program.cs
    ```

1. 기존 코드를 다음 코드 조각으로 바꿉니다. 

    코드는 앱의 전체 구조를 제공합니다. 코드의 주석을 검토하여 작동 방식을 이해해 보세요. 애플리케이션을 완성하려면 연습의 뒷부분에서 지정된 영역에 코드를 추가합니다. 

    ```csharp
    using Microsoft.Azure.Cosmos;
    using dotenv.net;
    
    string databaseName = "myDatabase"; // Name of the database to create or use
    string containerName = "myContainer"; // Name of the container to create or use
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    string cosmosDbAccountUrl = envVars["DOCUMENT_ENDPOINT"];
    string accountKey = envVars["ACCOUNT_KEY"];
    
    if (string.IsNullOrEmpty(cosmosDbAccountUrl) || string.IsNullOrEmpty(accountKey))
    {
        Console.WriteLine("Please set the DOCUMENT_ENDPOINT and ACCOUNT_KEY environment variables.");
        return;
    }
    
    // CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY
    
    
    try
    {
        // CREATE A DATABASE IF IT DOESN'T ALREADY EXIST
    
    
        // CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY
    
    
        // DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER
    
    
        // ADD THE ITEM TO THE CONTAINER
    
    
    }
    catch (CosmosException ex)
    {
        // Handle Cosmos DB-specific exceptions
        // Log the status code and error message for debugging
        Console.WriteLine($"Cosmos DB Error: {ex.StatusCode} - {ex.Message}");
    }
    catch (Exception ex)
    {
        // Handle general exceptions
        // Log the error message for debugging
        Console.WriteLine($"Error: {ex.Message}");
    }
    
    // This class represents a product in the Cosmos DB container
    public class Product
    {
        public string? id { get; set; }
        public string? name { get; set; }
        public string? description { get; set; }
    }
    ```

다음으로, 프로젝트의 지정된 영역에 코드를 추가하여 클라이언트, 데이터베이스, 컨테이너를 만들고 컨테이너에 샘플 항목을 추가합니다.

### 코드를 추가하여 클라이언트를 만들고 작업 수행하기 

1. **// CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY** 주석 다음에 있는 입력란에 다음 코드를 추가합니다. 이 코드는 Azure Cosmos DB 계정에 연결하는 데 사용되는 클라이언트를 정의합니다.

    ```csharp
    CosmosClient client = new(
        accountEndpoint: cosmosDbAccountUrl,
        authKeyOrResourceToken: accountKey
    );
    ```

    >참고: *Azure Identity* 라이브러리에서 **DefaultAzureCredential**을 사용하는 것이 가장 좋습니다. 이렇게 하려면 구독 설정 방법에 따라 Azure에서 몇 가지 추가 구성 요구 사항이 필요할 수 있습니다. 

1. **// CREATE A DATABASE IF IT DOESN'T ALREADY EXIST** 주석 다음에 있는 입력란에 다음 코드를 추가합니다. 

    ```csharp
    Database database = await client.CreateDatabaseIfNotExistsAsync(databaseName);
    Console.WriteLine($"Created or retrieved database: {database.Id}");
    ```

1. **// CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY** 주석 다음에 있는 입력란에 다음 코드를 추가합니다. 

    ```csharp
    Container container = await database.CreateContainerIfNotExistsAsync(
        id: containerName,
        partitionKeyPath: "/id"
    );
    Console.WriteLine($"Created or retrieved container: {container.Id}");
    ```

1. **// DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER** 주석 다음에 있는 입력란에 다음 코드를 추가합니다. 이는 컨테이너에 추가되는 항목을 정의합니다.

    ```csharp
    Product newItem = new Product
    {
        id = Guid.NewGuid().ToString(), // Generate a unique ID for the product
        name = "Sample Item",
        description = "This is a sample item in my Azure Cosmos DB exercise."
    };
    ```

1. **// ADD THE ITEM TO THE CONTAINER** 주석 다음에 있는 입력란에 다음 코드를 추가합니다. 

    ```csharp
    ItemResponse<Product> createResponse = await container.CreateItemAsync(
        item: newItem,
        partitionKey: new PartitionKey(newItem.id)
    );

    Console.WriteLine($"Created item with ID: {createResponse.Resource.id}");
    Console.WriteLine($"Request charge: {createResponse.RequestCharge} RUs");
    ```

1. 이제 코드가 완성되었으므로 진행률을 저장하고 **Ctrl + S**를 사용하여 파일을 저장하고, **Ctrl + Q**를 사용하여 편집기를 종료합니다.

1. Cloud Shell에서 다음 명령을 실행하여 프로젝트의 오류를 테스트합니다. 오류가 표시되면 편집기 에서 *Program.cs* 파일을 열고 누락된 코드 또는 붙여넣기 오류를 확인합니다.

    ```
    dotnet build
    ```

이제 프로젝트가 완료되었으므로 애플리케이션을 실행하고 Azure Portal에서 결과를 확인해야 합니다.

## 애플리케이션을 실행하고 결과를 확인합니다.

1. Cloud Shell에서 `dotnet run` 명령을 실행합니다. 출력은 다음 예시와 비슷해야 합니다.

    ```
    Created or retrieved database: myDatabase
    Created or retrieved container: myContainer
    Created item: c549c3fa-054d-40db-a42b-c05deabbc4a6
    Request charge: 6.29 RUs
    ```

1. Azure Portal에서 이전에 만든 Azure Cosmos DB 리소스로 이동합니다. 왼쪽 탐색에서 **Data Explorer**를 선택합니다. **Data Explorer**에서 **myDatabase**를 선택한 다음, **myContainer**를 확장합니다. **Items**을 선택하여 만든 항목을 볼 수 있습니다.

    ![Data Explorer에서 Items의 위치를 보여주는 스크린샷.](./media/01/cosmos-data-explorer.png)

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.
1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.
