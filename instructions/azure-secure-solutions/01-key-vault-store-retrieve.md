---
lab:
  topic: Secure solutions in Azure
  title: Azure Key Vault에서 비밀을 만들고 검색
  description: Azure CLI를 사용하여 키 자격 증명 모음을 만들고 비밀을 만들고 검색하는 방법과 프로그래밍 방식으로 키 자격 증명 모음을 만드는 방법을 알아보세요.
---

# Azure Key Vault에서 비밀을 만들고 검색

이 연습에서는 Azure Key Vault를 만들고, Azure CLI를 사용하여 비밀을 저장하고, 키 자격 증명 모음에서 비밀을 만들고 검색할 수 있는 .NET 콘솔 애플리케이션을 빌드해 보세요. 인증을 구성하고, 비밀을 프로그래밍 방식으로 관리하고, 작업이 완료되면 리소스를 정리하는 방법을 알아봅니다.  

이 연습에서 수행된 작업:

* Azure Key Vault 리소스 만들기
* Azure CLI를 사용하여 키 자격 증명 모음에 비밀 저장
* 비밀을 만들고 검색하기 위한 .NET 콘솔 앱 만들기
* 리소스 정리

이 연습을 완료하는 데 약 **30**분이 걸립니다.

## Azure Key Vault 리소스 만들기 및 비밀 추가

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
    keyVaultName=mykeyvaultname$RANDOM
    ```

1. 다음 명령을 실행하여 키 자격 증명 모음의 이름을 가져와서 기록합니다. 이후 연습에서 필요합니다.

    ```
    echo $keyVaultName
    ```

1. 다음 명령을 실행하여 Azure Key Vault 리소스를 만듭니다. 이 작업은 실행에 몇 분 정도 걸릴 수 있습니다.

    ```
    az keyvault create --name $keyVaultName \
        --resource-group $resourceGroup --location $location
    ```

### Microsoft Entra 사용자 이름에 역할 할당

비밀을 만들고 검색하려면 Microsoft Entra 사용자에게 **Key Vault Secrets Officer** 역할을 할당하세요. 이렇게 하면 사용자 계정에 비밀을 설정, 삭제, 나열할 수 있는 권한이 부여됩니다. 일반적인 시나리오에서는 **Key Vault Secrets Officer**를 한 그룹에 할당하고, **Key Vault Secrets User**(비밀을 가져와 나열할 수 있음)를 다른 그룹에 할당하여 만들기/읽기 작업을 구분할 수 있습니다.

1. 다음 명령을 실행하여 계정에서 **userPrincipalName**을 검색합니다. 이는 역할이 누구에게 할당될 것인지를 나타냅니다.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 다음 명령을 실행하여 키 자격 증명 모음의 리소스 ID를 검색합니다. 리소스 ID는 역할 할당의 범위를 특정 키 자격 증명 모음으로 설정합니다.

    ```
    resourceID=$(az keyvault show --resource-group $resourceGroup \
        --name $keyVaultName --query id --output tsv)
    ```

1. 다음 명령을 실행하여 **Key Vault Secrets Officer** 역할을 만들고 할당합니다.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Key Vault Secrets Officer" \
        --scope $resourceID
    ```

다음으로, 생성한 키 자격 증명 모음에 비밀을 추가합니다.

### Azure CLI를 사용하여 비밀 추가 및 검색

1. 다음 명령을 실행하여 비밀을 만들어 보세요. 

    ```
    az keyvault secret set --vault-name $keyVaultName \
        --name "MySecret" --value "My secret value"
    ```

1. 다음 명령을 실행하여 비밀을 검색하고, 비밀이 제대로 설정되었는지 확인하세요.

    ```
    az keyvault secret show --name "MySecret" --vault-name $keyVaultName
    ```

    이 명령은 일부 JSON을 반환합니다. 마지막 줄에는 일반 텍스트로 된 암호가 포함됩니다. 

    ```json
    "value": "My secret value"
    ```

## 비밀을 저장하고 검색하기 위한 .NET 콘솔 앱 만들기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. Cloud Shell에서 다음 단계를 수행합니다.

>**팁:** 위쪽 테두리를 끌어서 Cloud Shell의 크기를 조절하면 더 많은 정보와 코드가 표시됩니다. 최소화 및 최대화 단추를 사용하여 Cloud Shell과 기본 포털 인터페이스 사이를 전환할 수도 있습니다.

1. 다음 명령을 실행하여 프로젝트를 포함할 디렉터리를 만들고 해당 프로젝트 디렉터리로 이동합니다.

    ```
    mkdir keyvault
    cd keyvault
    ```

1. .NET 콘솔 애플리케이션을 만들어 보세요.

    ```
    dotnet new console
    ```

1. 다음 명령을 실행하여 프로젝트에 **Azure.Identity** 및 **Azure.Security.KeyVault.Secrets** 패키지를 추가합니다.

    ```
    dotnet add package Azure.Identity
    dotnet add package Azure.Security.KeyVault.Secrets
    ```

### 프로젝트의 시작 코드 추가

1. 애플리케이션을 편집하려면 Cloud Shell에서 다음 명령을 실행합니다.

    ```
    code Program.cs
    ```

1. 기존 콘텐츠를 다음 코드로 바꿉니다. **YOUR-KEYVAULT-NAME**을 실제 키 자격 증명 모음으로 바꿔야 합니다.

    ```csharp
    using Azure.Identity;
    using Azure.Security.KeyVault.Secrets;
    
    // Replace YOUR-KEYVAULT-NAME with your actual Key Vault name
    string KeyVaultUrl = "https://YOUR-KEYVAULT-NAME.vault.azure.net/";
    
    
    // ADD CODE TO CREATE A CLIENT
    
    
    
    // ADD CODE TO CREATE A MENU SYSTEM
    
    
    
    // ADD CODE TO CREATE A SECRET
    
    
    
    // ADD CODE TO LIST SECRETS
    
    
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장합니다.

### 애플리케이션을 완성하기 위한 코드 추가

이제 코드를 추가하여 애플리케이션을 완성할 차례입니다.

1. **// ADD CODE TO CREATE A CLIENT** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Configure authentication options for connecting to Azure Key Vault
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create the Key Vault client using the URL and authentication credentials
    var client = new SecretClient(new Uri(KeyVaultUrl), new DefaultAzureCredential(options));
    ```

1. **// ADD CODE TO CREATE A MENU SYSTEM** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    // Main application loop - continues until user types 'quit'
    while (true)
    {
        // Display menu options to the user
        Console.Clear();
        Console.WriteLine("\nPlease select an option:");
        Console.WriteLine("1. Create a new secret");
        Console.WriteLine("2. List all secrets");
        Console.WriteLine("Type 'quit' to exit");
        Console.Write("Enter your choice: ");
    
        // Read user input and convert to lowercase for easier comparison
        string? input = Console.ReadLine()?.Trim().ToLower();
        
        // Check if user wants to exit the application
        if (input == "quit")
        {
            Console.WriteLine("Goodbye!");
            break;
        }
    
        // Process the user's menu selection
        switch (input)
        {
            case "1":
                // Call the method to create a new secret
                await CreateSecretAsync(client);
                break;
            case "2":
                // Call the method to list all existing secrets
                await ListSecretsAsync(client);
                break;
            default:
                // Handle invalid input
                Console.WriteLine("Invalid option. Please enter 1, 2, or 'quit'.");
                break;
        }
    }
    ```

1. **// ADD CODE TO CREATE A SECRET** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    async Task CreateSecretAsync(SecretClient client)
    {
        try
        {
            Console.Clear();
            Console.WriteLine("\nCreating a new secret...");
            
            // Get the secret name from user input
            Console.Write("Enter secret name: ");
            string? secretName = Console.ReadLine()?.Trim();
    
            // Validate that the secret name is not empty
            if (string.IsNullOrEmpty(secretName))
            {
                Console.WriteLine("Secret name cannot be empty.");
                return;
            }
            
            // Get the secret value from user input
            Console.Write("Enter secret value: ");
            string? secretValue = Console.ReadLine()?.Trim();
    
            // Validate that the secret value is not empty
            if (string.IsNullOrEmpty(secretValue))
            {
                Console.WriteLine("Secret value cannot be empty.");
                return;
            }
    
            // Create a new KeyVaultSecret object with the provided name and value
            var secret = new KeyVaultSecret(secretName, secretValue);
            
            // Store the secret in Azure Key Vault
            await client.SetSecretAsync(secret);
    
            Console.WriteLine($"Secret '{secretName}' created successfully!");
            Console.WriteLine("Press Enter to continue...");
            Console.ReadLine();
        }
        catch (Exception ex)
        {
            // Handle any errors that occur during secret creation
            Console.WriteLine($"Error creating secret: {ex.Message}");
        }
    }
    ```

1. **// ADD CODE TO LIST SECRETS** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드와 주석을 반드시 검토해야 합니다.

    ```csharp
    async Task ListSecretsAsync(SecretClient client)
    {
        try
        {
            Console.Clear();
            Console.WriteLine("Listing all secrets in the Key Vault...");
            Console.WriteLine("----------------------------------------");
    
            // Get an async enumerable of all secret properties in the Key Vault
            var secretProperties = client.GetPropertiesOfSecretsAsync();
            bool hasSecrets = false;
    
            // Iterate through each secret property to retrieve full secret details
            await foreach (var secretProperty in secretProperties)
            {
                hasSecrets = true;
                try
                {
                    // Retrieve the actual secret value and metadata using the secret name
                    var secret = await client.GetSecretAsync(secretProperty.Name);
                    
                    // Display the secret information to the console
                    Console.WriteLine($"Name: {secret.Value.Name}");
                    Console.WriteLine($"Value: {secret.Value.Value}");
                    Console.WriteLine($"Created: {secret.Value.Properties.CreatedOn}");
                    Console.WriteLine("----------------------------------------");
                }
                catch (Exception ex)
                {
                    // Handle errors for individual secrets (e.g., access denied, secret not found)
                    Console.WriteLine($"Error retrieving secret '{secretProperty.Name}': {ex.Message}");
                    Console.WriteLine("----------------------------------------");
                }
            }
    
            // Inform user if no secrets were found in the Key Vault
            if (!hasSecrets)
            {
                Console.WriteLine("No secrets found in the Key Vault.");
            }
        }
        catch (Exception ex)
        {
            // Handle general errors that occur during the listing operation
            Console.WriteLine($"Error listing secrets: {ex.Message}");
        
        }
        Console.WriteLine("Press Enter to continue...");
        Console.ReadLine();
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

1. 다음 명령을 실행하여 콘솔 앱을 시작합니다. 앱은 해당 애플리케이션의 메뉴 시스템을 표시합니다. 

    ```
    dotnet run
    ```

1. 이 연습을 시작할 때 비밀을 생성했습니다. **2**를 입력하여 비밀을 검색하고 표시합니다.

1. **1**을 입력하고 비밀 이름과 값을 입력하여 새 비밀을 만듭니다.

1. 새로 추가된 내용을 보려면 비밀을 다시 나열하세요.

애플리케이션을 마치면 **quit**을 입력하세요.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.
1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.
