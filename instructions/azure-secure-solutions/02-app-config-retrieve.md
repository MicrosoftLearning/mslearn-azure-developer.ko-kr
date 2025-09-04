---
lab:
  topic: Secure solutions in Azure
  title: Azure App Configuration에서 구성 설정 검색하기
  description: 'Azure App Configuration 리소스를 만들고 Azure CLI를 사용하여 구성 정보를 설정하는 방법을 알아봅니다. 그런 다음, **ConfigurationBuilder**를 사용하여 애플리케이션의 설정을 검색합니다.'
---

# Azure App Configuration에서 구성 설정 검색하기

이 연습에서는 Azure App Configuration 리소스를 생성하고, Azure CLI를 사용하여 구성 설정을 저장한 다음, **ConfigurationBuilder**를 활용하여 구성 값을 가져오는 .NET 콘솔 애플리케이션을 구축해 보세요. 계층적 키를 사용하여 설정을 구성하고 클라우드 기반 구성 데이터에 액세스하기 위해 애플리케이션을 인증하는 방법을 알아봅니다.

이 연습에서 수행된 작업:

* Azure App Configuration 리소스 만들기
* 연결 문자열 구성 정보 저장
* 구성 정보를 검색하는 .NET 콘솔 앱 만들기
* 리소스 정리

이 연습을 완료하는 데 약 **15**분이 걸립니다.

## Azure App Configuration 리소스 만들기 및 구성 정보 추가

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
    appConfigName=appconfigname$RANDOM
    ```

1. 다음 명령을 실행하여 App Configuration 리소스의 이름을 가져옵니다. 이름을 기록해 두세요. 나중에 연습할 때 필요합니다.

    ```
    echo $appConfigName
    ```

1. 다음 명령을 실행하여 **Microsoft.AppConfiguration** 공급자가 구독에 등록되어 있는지 확인합니다.

    ```
    az provider register --namespace Microsoft.AppConfiguration
    ```

1. 등록이 완료되려면 몇 분 정도 걸릴 수 있습니다. 다음 명령을 실행하여 등록 상태를 확인하세요. 결과가 **Registered**로 반환되면 다음 단계로 진행합니다.

    ```
    az provider show --namespace Microsoft.AppConfiguration --query "registrationState"
    ```

1. 다음 명령을 실행하여 Azure App Configuration 리소스를 만듭니다. 이 작업은 실행에 몇 분 정도 걸릴 수 있습니다.

    ```
    az appconfig create --location $location \
        --name $appConfigName \
        --resource-group $resourceGroup
        --sku Free
    ```

    >**팁:** **무료** SKU 값을 사용하는 할당량 제한으로 인해 AppConfig 리소스를 만드는 데 문제가 있는 경우, 대신 **개발자**를 사용하세요.
    

### Microsoft Entra 사용자 이름에 역할 할당

구성 정보를 검색하려면 Microsoft Entra 사용자에게 **App Configuration Data Reader** 역할을 할당해야 합니다. 

1. 다음 명령을 실행하여 계정에서 **userPrincipalName**을 검색합니다. 이는 역할이 누구에게 할당될 것인지를 나타냅니다.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 다음 명령을 실행하여 App Configuration 서비스의 리소스 ID를 검색합니다. 리소스 ID는 역할 할당의 범위를 설정합니다.

    ```
    resourceID=$(az appconfig show --resource-group $resourceGroup \
        --name $appConfigName --query id --output tsv)
    ```

1. 다음 명령을 실행하여 **App Configuration 데이터 Reader** 역할을 만들고 할당합니다.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "App Configuration Data Reader" \
        --scope $resourceID
    ```

다음으로 자리 표시자 연결 문자열을 App Configuration에 추가합니다.

### Azure CLI를 사용하여 구성 정보 추가

Azure App Configuration에서 **Dev:conStr**과 같은 키는 계층적 키 또는 네임스페이스 키입니다. 콜론(:)은 논리적 계층을 만드는 구분 기호 역할을 합니다.

* **Dev**는 네임스페이스 또는 환경 접두사를 나타냅니다(이 구성이 개발 환경을 위한 것임을 나타냄).
* **conStr**은 구성 이름을 나타냅니다.

이러한 계층적 구조를 사용하면 환경, 기능, 애플리케이션 구성 요소별로 구성 설정을 구성하여 관련 설정을 보다 쉽게 ​​관리하고 검색할 수 있습니다.

다음 명령을 실행하여 자리 표시자 연결 문자열을 저장합니다. 

```
az appconfig kv set --name $appConfigName \
    --key Dev:conStr \
    --value connectionString \
    --yes
```

이 명령은 일부 JSON을 반환합니다. 마지막 줄에는 일반 텍스트로 된 값이 포함됩니다. 

```json
"value": "connectionString"
```

## 구성 정보를 검색하는 .NET 콘솔 앱 만들기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. Cloud Shell에서 다음 단계를 수행합니다.

>**팁:** 위쪽 테두리를 끌어서 Cloud Shell의 크기를 조절하면 더 많은 정보와 코드가 표시됩니다. 최소화 및 최대화 단추를 사용하여 Cloud Shell과 기본 포털 인터페이스 사이를 전환할 수도 있습니다.

1. 다음 명령을 실행하여 프로젝트를 포함할 디렉터리를 만들고 해당 프로젝트 디렉터리로 이동합니다.

    ```
    mkdir appconfig
    cd appconfig
    ```

1. .NET 콘솔 애플리케이션을 만들어 보세요.

    ```
    dotnet new console
    ```

1. 다음 명령을 실행하여 프로젝트에 **Azure.Identity** 및 **Microsoft.Extensions.Configuration.AzureAppConfiguration** 패키지를 추가합니다.

    ```
    dotnet add package Azure.Identity
    dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
    ```

### 프로젝트용 코드 추가

1. 애플리케이션을 편집하려면 Cloud Shell에서 다음 명령을 실행합니다.

    ```
    code Program.cs
    ```

1. 기존 콘텐츠를 다음 코드로 바꿉니다. **YOUR_APP_CONFIGURATION_NAME**을 이전에 기록해 둔 이름으로 바꾸고 코드의 주석을 꼼꼼히 읽어보세요.

    ```csharp
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.Configuration.AzureAppConfiguration;
    using Azure.Identity;
    
    // Set the Azure App Configuration endpoint, replace YOUR_APP_CONFIGURATION_NAME
    // with the name of your actual App Configuration service
    
    string endpoint = "https://YOUR_APP_CONFIGURATION_NAME.azconfig.io"; 
    
    // Configure which authentication methods to use
    // DefaultAzureCredential tries multiple auth methods automatically
    DefaultAzureCredentialOptions credentialOptions = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a configuration builder to combine multiple config sources
    var builder = new ConfigurationBuilder();
    
    // Add Azure App Configuration as a source
    // This connects to Azure and loads configuration values
    builder.AddAzureAppConfiguration(options =>
    {
        
        options.Connect(new Uri(endpoint), new DefaultAzureCredential(credentialOptions));
    });
    
    // Build the final configuration object
    try
    {
        var config = builder.Build();
        
        // Retrieve a configuration value by key name
        Console.WriteLine(config["Dev:conStr"]);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error connecting to Azure App Configuration: {ex.Message}");
    }
    ```

1. **Ctrl+S**를 눌러 파일을 저장한 다음, **Ctrl+Q**를 눌러 편집기를 종료합니다.

## Azure에 로그인하고 앱 실행

1. Cloud Shell에서 다음 명령을 입력하여 Azure에 로그인합니다.

    ```
    az login
    ```

    **<font color="red">Cloud Shell 세션이 이미 인증되었더라도 Azure에 로그인해야 합니다.</font>**

    > **참고**: 대부분의 시나리오에서는 *az login*을 사용하는 것만으로도 충분합니다. 그러나 여러 테넌트에 구독이 있는 경우 *--tenant* 매개 변수를 사용하여 테넌트 지정해야 할 수 있습니다. 자세한 내용은 [Azure CLI를 사용하여 대화형으로 Azure에 로그인](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)을 참조하세요.

1. 다음 명령을 실행하여 콘솔 앱을 시작합니다. 앱은 연습 앞부분에서 **Dev:conStr** 설정에 할당한 **connectionString** 값을 표시합니다.

    ```
    dotnet run
    ```

    앱은 연습 앞부분에서 **Dev:conStr** 설정에 할당한 **connectionString** 값을 표시합니다.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 범위를 벗어난 기존 리소스는 
