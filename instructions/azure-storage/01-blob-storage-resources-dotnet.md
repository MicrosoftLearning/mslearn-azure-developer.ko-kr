---
lab:
  topic: Azure Storage
  title: .NET 클라이언트 라이브러리를 사용하여 Blob Storage 리소스 만들기
  description: 'Azure Storage .NET 클라이언트 라이브러리를 사용하여 컨테이너를 만들고, Blob을 업로드 및 나열하고, 컨테이너를 삭제하는 방법을 알아봅니다.'
---

# .NET 클라이언트 라이브러리를 사용하여 Blob Storage 리소스 만들기

이 연습에서는 Azure Storage 계정을 만들고 Azure Storage Blob 클라이언트 라이브러리를 사용하여 .NET 콘솔 애플리케이션을 빌드하여 컨테이너를 만들고, Blob Storage에 파일을 업로드하고, Blob을 나열하여, 파일을 다운로드해 보세요. Azure에서 인증하는 방법, Blob Storage 작업을 프로그래밍 방식으로 수행하는 방법, Azure Portal에서 결과를 확인하는 방법을 알아봅니다.

이 연습에서 수행된 작업:

* Azure 리소스 준비
* 데이터를 만들고 다운로드하는 콘솔 앱 만들기
* 앱 실행 및 결과 확인
* 리소스 정리

이 연습을 완료하는 데 약 **30**분이 걸립니다.

## Azure Storage 계정 만들기

이 연습 섹션에서는 Azure CLI를 사용하여 Azure에서 필요한 리소스를 만듭니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.

1. 페이지 상단의 검색 창 오른쪽에 있는 **[\>_]** 단추를 사용하여 Azure Portal에서 새 Cloud Shell을 만들고 ***Bash*** 환경을 선택합니다. Cloud Shell은 다음과 같이 Azure Portal 아래쪽 창에 명령줄 인터페이스를 제공합니다. 파일을 보관할 스토리지 계정을 선택하라는 프롬프트가 표시되면 **스토리지 계정 필요 없음**, 구독을 차례로 선택한 다음, **적용**을 선택합니다.

    > **참고**: 이전에 *PowerShell* 환경을 사용하는 Cloud Shell을 만든 경우 ***Bash***로 전환합니다.

1. Cloud Shell 도구 모음의 **설정** 메뉴에서 **클래식 버전으로 이동**을 선택합니다(코드 편집기를 사용하는 데 필요).

1. 이 연습에 필요한 리소스에 대한 리소스 그룹을 만듭니다. **myResourceGroup**을 리소스 그룹에 사용하려는 이름으로 바꾸세요. 필요한 경우, **eastus2**를 근처 지역으로 바꿀 수 있습니다. 사용하려는 리소스 그룹이 이미 있는 경우, 다음 단계를 진행하세요.

    ```
    az group create --location eastus2 --name myResourceGroup
    ```

1. 대부분의 명령은 고유한 이름이 필요하며 동일한 매개 변수를 사용합니다. 일부 변수를 만들어 두면 리소스를 만드는 명령에 필요한 변경 내용을 줄일 수 있습니다. 다음 명령을 실행하여 필요한 변수를 만들 수 있습니다. **myResourceGroup**을 이 연습에 사용하는 이름으로 바꾸세요.

    ```
    resourceGroup=myResourceGroup
    location=eastus
    accountName=storageacct$RANDOM
    ```

1. 다음 명령을 실행하여 Azure Storage DB 계정을 만드세요. 각 계정 이름은 고유해야 합니다. 첫 번째 명령은 스토리지 계정의 고유한 이름을 가진 변수를 만듭니다. **echo** 명령의 출력에서 ​​계정 이름을 기록합니다. 

    ```
    az storage account create --name $accountName \
        --resource-group $resourceGroup \
        --location $location \
        --sku Standard_LRS 
    
    echo $accountName
    ```

### Microsoft Entra 사용자 이름에 역할 할당

앱에서 리소스와 항목을 만들 수 있도록 하려면 Microsoft Entra 사용자에게 **Storage Blob 데이터 소유자** 역할을 할당하세요. Cloud Shell에서 다음 단계를 수행합니다.

>**팁:** 위쪽 테두리를 끌어서 Cloud Shell의 크기를 조절하면 더 많은 정보와 코드가 표시됩니다. 최소화 및 최대화 단추를 사용하여 Cloud Shell과 기본 포털 인터페이스 사이를 전환할 수도 있습니다.

1. 다음 명령을 실행하여 계정에서 **userPrincipalName**을 검색합니다. 이는 역할이 누구에게 할당될 것인지를 나타냅니다.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 다음 명령을 실행하여 스토리지 계정의 리소스 ID를 검색합니다. 리소스 ID는 역할 할당의 범위를 특정 네임스페이스로 설정합니다.

    ```
    resourceID=$(az storage account show --name $accountName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. 다음 명령을 실행하여 **Storage Blob 데이터 소유자** 역할을 만들고 할당합니다. 이 역할은 컨테이너 및 항목을 관리할 수 있는 권한을 부여합니다.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Blob Data Owner" \
        --scope $resourceID
    ```

## 컨테이너 및 항목을 만드는 .NET 콘솔 앱 만들기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. Cloud Shell에서 다음 단계를 수행합니다.

1. 다음 명령을 실행하여 프로젝트를 포함할 디렉터리를 만들고 해당 프로젝트 디렉터리로 이동합니다.

    ```
    mkdir azstor
    cd azstor
    ```

1. .NET 콘솔 애플리케이션을 만들어 보세요.

    ```
    dotnet new console
    ```

1. 다음 명령을 실행하여 애플리케이션에 필요한 패키지를 추가하세요.

    ```
    dotnet add package Azure.Storage.Blobs
    dotnet add package Azure.Identity
    ```

1. 다음 명령을 실행하여 프로젝트에 **data** 폴더를 만듭니다. 

    ```
    mkdir data
    ```

이제 프로젝트에 코드를 추가할 차례입니다.

### 프로젝트의 시작 코드 추가

1. 애플리케이션을 편집하려면 Cloud Shell에서 다음 명령을 실행합니다.

    ```
    code Program.cs
    ```

1. 기존 콘텐츠를 다음 코드로 바꿉니다. 코드의 주석을 반드시 검토하세요.

    ```csharp
    using Azure.Storage.Blobs;
    using Azure.Storage.Blobs.Models;
    using Azure.Identity;
    
    Console.WriteLine("Azure Blob Storage exercise\n");
    
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Run the examples asynchronously, wait for the results before proceeding
    await ProcessAsync();
    
    Console.WriteLine("\nPress enter to exit the sample application.");
    Console.ReadLine();
    
    async Task ProcessAsync()
    {
        // CREATE A BLOB STORAGE CLIENT
        
    
    
        // CREATE A CONTAINER
        
    
    
        // CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE
        
    
    
        // UPLOAD THE FILE TO BLOB STORAGE
        
    
    
        // LIST BLOBS IN THE CONTAINER
        
    
    
        // DOWNLOAD THE BLOB TO A LOCAL FILE
        
    
    }
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장하고 다음 단계로 넘어가세요.


## 프로젝트를 완료하는 코드 추가

연습의 나머지 부분에서는 지정된 영역에 코드를 추가하여 전체 애플리케이션을 만듭니다. 

1. **// CREATE A BLOB STORAGE CLIENT** 주석을 찾은 다음, 주석 바로 아래에 다음 코드를 추가합니다. **BlobServiceClient**는 스토리지 계정에서 컨테이너와 Blob을 관리하기 위한 기본 진입점 역할을 합니다. 클라이언트는 인증을 위해 *DefaultAzureCredential*을 사용합니다. **YOUR_ACCOUNT_NAME**을 이전에 기록해 둔 이름으로 바꿔야 합니다.

    ```csharp
    // Create a credential using DefaultAzureCredential with configured options
    string accountName = "YOUR_ACCOUNT_NAME"; // Replace with your storage account name
    
    // Use the DefaultAzureCredential with the options configured at the top of the program
    DefaultAzureCredential credential = new DefaultAzureCredential(options);
    
    // Create the BlobServiceClient using the endpoint and DefaultAzureCredential
    string blobServiceEndpoint = $"https://{accountName}.blob.core.windows.net";
    BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), credential);
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장하고 다음 단계로 넘어가세요.

1. **// CREATE A CONTAINER** 주석을 찾은 다음, 주석 바로 아래에 다음 코드를 추가합니다. 컨테이너를 만드는 과정에는 **BlobServiceClient** 클래스의 인스턴스를 만든 다음, **CreateBlobContainerAsync** 메서드를 호출하여 스토리지 계정에 컨테이너를 만드는 작업이 포함됩니다. 컨테이너 이름을 고유하게 만들기 위해 GUID 값을 컨테이너 이름에 추가합니다. 컨테이너가 이미 있으면 **CreateBlobContainerAsync** 메서드는 실패합니다.

    ```csharp
    // Create a unique name for the container
    string containerName = "wtblob" + Guid.NewGuid().ToString();
    
    // Create the container and return a container client object
    Console.WriteLine("Creating container: " + containerName);
    BlobContainerClient containerClient = 
        await blobServiceClient.CreateBlobContainerAsync(containerName);
    
    // Check if the container was created successfully
    if (containerClient != null)
    {
        Console.WriteLine("Container created successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("Failed to create the container, exiting program.");
        return;
    }
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장하고 다음 단계로 넘어가세요.

1. **// CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE** 주석을 찾은 다음, 주석 바로 아래에 다음 코드를 추가합니다. 이렇게 하면 컨테이너에 업로드되는 데이터 디렉터리에 파일이 만들어집니다.

    ```csharp
    // Create a local file in the ./data/ directory for uploading and downloading
    Console.WriteLine("Creating a local file for upload to Blob storage...");
    string localPath = "./data/";
    string fileName = "wtfile" + Guid.NewGuid().ToString() + ".txt";
    string localFilePath = Path.Combine(localPath, fileName);
    
    // Write text to the file
    await File.WriteAllTextAsync(localFilePath, "Hello, World!");
    Console.WriteLine("Local file created, press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장하고 다음 단계로 넘어가세요.

1. **// UPLOAD THE FILE TO BLOB STORAGE** 주석을 찾은 다음, 주석 바로 아래에 다음 코드를 추가합니다. 이 코드는 이전 섹션에서 만든 컨테이너에서 **GetBlobClient** 메서드를 호출하여 **BlobClient** 개체에 대한 참조를 가져옵니다. 그런 다음, **UploadAsync** 메서드를 사용하여 생성된 로컬 파일을 업로드합니다. 이 메서드는 Blob이 없는 경우 만들고, Blob이 있는 경우 덮어씁니다.

    ```csharp
    // Get a reference to the blob and upload the file
    BlobClient blobClient = containerClient.GetBlobClient(fileName);
    
    Console.WriteLine("Uploading to Blob storage as blob:\n\t {0}", blobClient.Uri);
    
    // Open the file and upload its data
    using (FileStream uploadFileStream = File.OpenRead(localFilePath))
    {
        await blobClient.UploadAsync(uploadFileStream);
        uploadFileStream.Close();
    }
    
    // Verify if the file was uploaded successfully
    bool blobExists = await blobClient.ExistsAsync();
    if (blobExists)
    {
        Console.WriteLine("File uploaded successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("File upload failed, exiting program..");
        return;
    }
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장하고 다음 단계로 넘어가세요.

1. **// LIST BLOBS IN THE CONTAINER** 주석을 찾은 다음, 주석 바로 아래에 다음 코드를 추가합니다. **GetBlobsAsync** 메서드를 사용하여 컨테이너에 있는 Blob을 나열합니다. 이 경우 하나의 Blob만 컨테이너에 추가되므로 나열 작업은 하나의 Blob만 반환합니다. 

    ```csharp
    Console.WriteLine("Listing blobs in container...");
    await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
    {
        Console.WriteLine("\t" + blobItem.Name);
    }
    
    Console.WriteLine("Press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장하고 다음 단계로 넘어가세요.

1. **// DOWNLOAD THE BLOB TO A LOCAL FILE** 주석을 찾은 다음, 주석 바로 아래에 다음 코드를 추가합니다. 이 코드는 **DownloadAsync** 메서드를 사용하여 이전에 생성된 Blob을 로컬 파일 시스템에 다운로드합니다. 예제 코드는 로컬 파일 시스템에서 두 파일을 모두 볼 수 있도록 Blob 이름에 “DOWNLOADED” 접미사를 추가합니다. 

    ```csharp
    // Adds the string "DOWNLOADED" before the .txt extension so it doesn't 
    // overwrite the original file
    
    string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");
    
    Console.WriteLine("Downloading blob to: {0}", downloadFilePath);
    
    // Download the blob's contents and save it to a file
    BlobDownloadInfo download = await blobClient.DownloadAsync();
    
    using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath))
    {
        await download.Content.CopyToAsync(downloadFileStream);
    }
    
    Console.WriteLine("Blob downloaded successfully to: {0}", downloadFilePath);
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

1. 왼쪽 탐색에서 **> 데이터 스토리지**를 확장하고 **컨테이너**를 선택하세요.

1. 애플리케이션이 만든 컨테이너를 선택하면 업로드된 Blob을 볼 수 있습니다.

1. 아래 두 명령을 실행하여 **data** 디렉터리로 변경하고 업로드 및 다운로드된 파일을 나열합니다.

    ```
    cd data
    ls
    ```

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.
1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.

