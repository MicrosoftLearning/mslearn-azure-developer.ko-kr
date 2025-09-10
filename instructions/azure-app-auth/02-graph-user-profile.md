---
lab:
  topic: Azure authentication and authorization
  title: Microsoft Graph SDK를 사용하여 사용자 프로필 정보 검색하기
  description: Microsoft Graph에서 사용자 프로필 정보 검색 방법을 알아보세요.
---

# Microsoft Graph SDK를 사용하여 사용자 프로필 정보 검색하기

이 연습에서는 Microsoft Entra ID로 인증하고 액세스 토큰을 요청하는 .NET 애플리케이션을 만든 후, Microsoft Graph API를 호출하여 사용자 프로필 정보를 검색하고 이를 표시해 보세요. 애플리케이션에서 권한을 구성하고 Microsoft Graph와 상호 작용하는 방법을 알아봅니다.

이 연습에서 수행된 작업:

* Microsoft ID 플랫폼을 사용하여 애플리케이션 등록
* 대화형 인증을 구현하고 **GraphServiceClient** 클래스를 사용하여 사용자 프로필 정보를 검색하는 .NET 콘솔 애플리케이션을 만들어 보세요.

이 연습을 완료하는 데 약 **15**분이 걸립니다.

## 시작하기 전에

연습을 완료하려면 다음 사항이 필요합니다.

* Azure 구독 아직 계정이 없다면 [등록](https://azure.microsoft.com/)할 수 있습니다.

* [지원되는 플랫폼](https://code.visualstudio.com/docs/supporting/requirements#_platforms) 중 하나인 [Visual Studio Code](https://code.visualstudio.com/).

* [.NET 8](https://dotnet.microsoft.com/en-us/download/dotnet/8.0) 이상

* Visual Studio Code용 [C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit).

## 새 애플리케이션 등록

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.

1. 포털에서 **앱 등록**을 검색하여 선택합니다. 

1. **+ 신규 등록**을 선택하고 **애플리케이션 등록** 페이지가 나타나면 애플리케이션의 등록 정보를 입력하세요.

    | 필드 | 값 |
    |--|--|
    | **이름** | `myGraphApplication`을 입력합니다.  |
    | **지원되는 계정 유형** | **이 조직 디렉터리의 계정만** 선택 |
    | **리디렉션 URI(선택 사항)** | **퍼블릭 클라이언트/네이티브(모바일 및 데스크톱)** 를 선택하고 오른쪽에 있는 상자에 `http://localhost`를 입력합니다. |

1. **등록**을 선택합니다. 이 작동하려면 다음 사항이 요구됩니다.Features and licenses for Microsoft Entra가 앱에 고유한 애플리케이션(클라이언트) ID를 할당하며, 애플리케이션의 **개요** 페이지가 표시됩니다. 

1. **개요** 페이지의 **필수 사항** 섹션에서 **애플리케이션(클라이언트) ID** 및 **디렉터리(테넌트) ID**를 기록합니다. 해당 정보는 애플리케이션에 필요합니다.

    ![복사할 필드의 위치를 보여 주는 스크린샷](./media/01-app-directory-id-location.png)
 
## 메시지를 보내고 받기 위한 .NET 콘솔 앱 만들기

이제 필요한 리소스가 Azure에 배포되었으므로 다음 단계는 콘솔 애플리케이션을 설정하는 것입니다. 다음 단계는 로컬 환경에서 수행됩니다.

1. 프로젝트에서 **graphapp**이라는 이름의 폴더 또는 원하는 이름을 만듭니다.

1. **Visual Studio Code**를 실행하고 **파일 > 폴더 열기...** 를 선택한 다음, 프로젝트 폴더를 선택합니다.

1. 터미널을 열려면 **보기 > 터미널**을 선택하세요.

1. VS Code 터미널에서 다음 명령을 실행하여 .NET 콘솔 애플리케이션을 만듭니다.

    ```
    dotnet new console
    ```

1. 다음 명령을 실행하여 프로젝트에 **Azure.Identity**, **Microsoft.Graph**, **dotenv.net** 패키지를 추가합니다.

    ```
    dotnet add package Azure.Identity
    dotnet add package Microsoft.Graph
    dotnet add package dotenv.net
    ```

### 콘솔 애플리케이션 구성

이 섹션에서는 이전에 기록한 비밀을 보관할 **.env** 파일을 만들고 편집합니다. 

1. **파일 > 새 파일...** 을 선택하고 프로젝트 폴더에 *.env*라는 이름의 파일을 만듭니다.

1. **.env** 파일을 열고 다음 코드를 추가합니다. **YOUR_CLIENT_ID** 및 **YOUR_TENANT_ID**를 이전에 기록한 값으로 바꾸세요.

    ```
    CLIENT_ID="YOUR_CLIENT_ID"
    TENANT_ID="YOUR_TENANT_ID"
    ```

1. **Ctrl+S** 키를 눌러 파일을 저장합니다.

### 프로젝트의 시작 코드 추가

1. *Program.cs* 파일을 열고 기존 콘텐츠를 다음 코드로 바꾸세요. 코드의 주석을 반드시 검토하세요.

    ```csharp
    using Microsoft.Graph;
    using Azure.Identity;
    using dotenv.net;
    
    // Load environment variables from .env file (if present)
    DotEnv.Load();
    var envVars = DotEnv.Read();
    
    // Read Azure AD app registration values from environment
    string clientId = envVars["CLIENT_ID"];
    string tenantId = envVars["TENANT_ID"];
    
    // Validate that required environment variables are set
    if (string.IsNullOrEmpty(clientId) || string.IsNullOrEmpty(tenantId))
    {
        Console.WriteLine("Please set CLIENT_ID and TENANT_ID environment variables.");
        return;
    }
    
    // ADD CODE TO DEFINE SCOPE AND CONFIGURE AUTHENTICATION
    
    
    
    // ADD CODE TO CREATE GRAPH CLIENT AND RETRIEVE USER PROFILE
    
    
    ```

1. **Ctrl+S**를 눌러 변경 내용을 저장합니다.

### 애플리케이션을 완성하기 위한 코드 추가

1. **// ADD CODE TO DEFINE SCOPE AND CONFIGURE AUTHENTICATION** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드의 주석을 반드시 검토하세요.

    ```csharp
    // Define the Microsoft Graph permission scopes required by this app
    var scopes = new[] { "User.Read" };
    
    // Configure interactive browser authentication for the user
    var options = new InteractiveBrowserCredentialOptions
    {
        ClientId = clientId, // Azure AD app client ID
        TenantId = tenantId, // Azure AD tenant ID
        RedirectUri = new Uri("http://localhost") // Redirect URI for auth flow
    };
    var credential = new InteractiveBrowserCredential(options);
    ```

1. **// ADD CODE TO CREATE GRAPH CLIENT AND RETRIEVE USER PROFILE** 주석을 찾아 주석 바로 뒤에 다음 코드를 추가합니다. 코드의 주석을 반드시 검토하세요.

    ```csharp
    // Create a Microsoft Graph client using the credential
    var graphClient = new GraphServiceClient(credential);
    
    // Retrieve and display the user's profile information
    Console.WriteLine("Retrieving user profile...");
    await GetUserProfile(graphClient);
    
    // Function to get and print the signed-in user's profile
    async Task GetUserProfile(GraphServiceClient graphClient)
    {
        try
        {
            // Call Microsoft Graph /me endpoint to get user info
            var me = await graphClient.Me.GetAsync();
            Console.WriteLine($"Display Name: {me?.DisplayName}");
            Console.WriteLine($"Principal Name: {me?.UserPrincipalName}");
            Console.WriteLine($"User Id: {me?.Id}");
        }
        catch (Exception ex)
        {
            // Print any errors encountered during the call
            Console.WriteLine($"Error retrieving profile: {ex.Message}");
        }
    }
    ```

1. **Ctrl+S** 키를 눌러 파일을 저장합니다.

## 애플리케이션 실행

이제 앱이 완성되었으므로 실행할 차례입니다. 

1. 다음 명령을 실행하여 애플리케이션을 시작합니다.

    ```
    dotnet run
    ```

1. 앱에 기본 브라우저가 열리면서 인증할 계정을 선택하라는 메시지가 표시됩니다. 나열된 계정이 여러 개이면 앱에서 사용하는 테넌트와 연결된 계정이 선택됩니다.

1. 등록된 앱에 처음 인증하는 경우, 앱에 로그인하고 프로필을 읽고, 액세스 권한을 부여한 데이터에 대한 액세스를 유지하도록 승인할지 묻는 **권한 요청** 알림이 표시됩니다. **수락**을 선택합니다.

    ![권한 요청 알림을 보여 주는 스크린샷](./media/01-granting-permission.png)

1. 콘솔에 아래 예제와 비슷한 결과가 표시됩니다.

    ```
    Retrieving user profile...
    Display Name: <Your account display name>
    Principal Name: <Your principal name>
    User Id: 9f5...
    ```

1. 두 번째로 애플리케이션을 시작하면 더 이상 **요청된 권한** 알림을 받지 못하게 됩니다. 이전에 부여한 사용 권한은 캐시되었습니다.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.
1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.
