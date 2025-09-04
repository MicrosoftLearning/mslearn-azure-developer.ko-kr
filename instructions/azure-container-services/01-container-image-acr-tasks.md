---
lab:
  topic: Azure container services
  title: Azure Container Registry 작업으로 컨테이너 이미지 빌드 및 실행하기
  description: Azure Container Registry 작업으로 컨테이너 이미지를 빌드하고 실행하기 위해 Azure CLI 명령을 사용하는 방법을 알아보세요.
---

# Azure Container Registry 작업으로 컨테이너 이미지 빌드 및 실행하기

이 연습에서는 Azure CLI를 사용하여 애플리케이션 코드를 기반으로 컨테이너 이미지를 빌드하고, Azure Container Registry에 푸시하세요. 이 연습을 통해 앱을 컨테이너화할 준비를 하고, ACR 인스턴스를 생성하며, 생성한 컨테이너 이미지를 Azure에 저장하는 방법을 학습하게 됩니다.

이 연습에서 수행된 작업:

* Azure 컨테이너 레지스트리 리소스 만들기
* Dockerfile에서 이미지 빌드 및 푸시
* 결과 확인
* Azure Container Registry에서 이미지 실행

이 연습을 완료하는 데 약 **20**분이 걸립니다.

## Azure 컨테이너 레지스트리 리소스 만들기

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.

1. 페이지 상단의 검색 창 오른쪽에 있는 **[\>_]** 단추를 사용하여 Azure Portal에서 새 Cloud Shell을 만들고 ***Bash*** 환경을 선택합니다. Cloud Shell은 다음과 같이 Azure Portal 아래쪽 창에 명령줄 인터페이스를 제공합니다. 파일을 보관할 스토리지 계정을 선택하라는 프롬프트가 표시되면 **스토리지 계정 필요 없음**, 구독을 차례로 선택한 다음, **적용**을 선택합니다.

    > **참고**: 이전에 *PowerShell* 환경을 사용하는 Cloud Shell을 만든 경우 ***Bash***로 전환합니다.

1. 이 연습에 필요한 리소스에 대한 리소스 그룹을 만듭니다. **myResourceGroup**을 리소스 그룹에 사용하려는 이름으로 바꾸세요. 필요한 경우 **eastus**를 근처 지역으로 바꿀 수 있습니다. 사용하려는 리소스 그룹이 이미 있는 경우, 다음 단계를 진행하세요.

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. 다음 명령을 실행하여 기본 컨테이너 레지스트리를 만듭니다. 레지스트리 이름은 Azure 내에서 고유해야 하며, 5-50자의 영숫자만 포함해야 합니다. **myResourceGroup**을 이전에 사용한 이름으로 바꾸고, **myContainerRegistry**를 고유한 값으로 바꾸세요.

    ```bash
    az acr create --resource-group myResourceGroup \
        --name myContainerRegistry --sku Basic
    ```

    > **참고:** 명령은 개발자가 Azure Container Registry에 대해 학습하는 데 사용할 수 있는 비용 최적화 옵션인 기본 레지스트리를 만듭니다.**

## Dockerfile에서 이미지 빌드 및 푸시

다음으로, Dockerfile을 기반으로 이미지를 빌드하고 푸시합니다.

1. 다음 명령을 실행하여 Dockerfile을 만듭니다. Dockerfile에는 Microsoft Container Registry에서 호스트되는 *hello-world* 이미지를 참조하는 단일 줄이 포함되어 있습니다.

    ```bash
    echo FROM mcr.microsoft.com/hello-world > Dockerfile
    ```

1. 다음 **az acr build** 명령을 실행하면 이미지가 빌드됩니다. 이미지가 성공적으로 빌드된 후, 레지스트리에 푸시됩니다. **myContainerRegistry**를 앞에서 만든 이름으로 바꾸세요.

    ```bash
    az acr build --image sample/hello-world:v1  \
        --registry myContainerRegistry \
        --file Dockerfile .
    ```

    다음은 최종 결과의 마지막 몇 줄을 보여 주는 이전 명령의 출력에 대한 축약된 샘플입니다. *repository* 필드에 *sample/hello-word* 이미지가 나열되어 있는 것을 볼 수 있습니다.

    ```
    - image:
        registry: myContainerRegistry.azurecr.io
        repository: sample/hello-world
        tag: v1
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      runtime-dependency:
        registry: mcr.microsoft.com
        repository: hello-world
        tag: latest
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      git: {}
    
    
    Run ID: cf1 was successful after 11s
    ```

## 결과 확인

1. 다음 명령을 실행하여 레지스트리의 리포지토리 목록을 표시합니다. **myContainerRegistry**를 앞에서 만든 이름으로 바꾸세요.

    ```bash
    az acr repository list --name myContainerRegistry --output table
    ```

    출력:

    ```
    Result
    ----------------
    sample/hello-world
    ```

1. **sample/hello-world** 리포지토리의 태그를 나열하려면 다음 명령을 실행하세요. **myContainerRegistry**는 앞에서 사용한 이름으로 바꾸세요.

    ```bash
    az acr repository show-tags --name myContainerRegistry \
        --repository sample/hello-world --output table
    ```

    출력:

    ```
    Result
    --------
    v1
    ```

## ACR에서 이미지 실행

1. **az acr run** 명령을 사용하여 컨테이너 레지스트리에서 *sample/hello-world:v1* 컨테이너 이미지를 실행합니다. 다음 예제에서는 **$Registry**를 사용하여 명령을 실행하는 레지스트리를 지정합니다. **myContainerRegistry**는 앞에서 사용한 이름으로 바꾸세요.

    ```bash
    az acr run --registry myContainerRegistry \
        --cmd '$Registry/sample/hello-world:v1' /dev/null
    ```

    이 예에서 **cmd** 매개 변수는 기본 구성에서 컨테이너를 실행하지만 **cmd**는 다른 **docker run** 매개 변수 또는 다른 **docker** 명령도 지원합니다. 

    다음 샘플이 축약되어 출력됩니다.

    ```
    Packing source code into tar to upload...
    Uploading archived source code from '/tmp/run_archive_ebf74da7fcb04683867b129e2ccad5e1.tar.gz'...
    Sending context (1.855 KiB) to registry: mycontainerre...
    Queued a run with ID: cab
    Waiting for an agent...
    2019/03/19 19:01:53 Using acb_vol_60e9a538-b466-475f-9565-80c5b93eaa15 as the home volume
    2019/03/19 19:01:53 Creating Docker network: acb_default_network, driver: 'bridge'
    2019/03/19 19:01:53 Successfully set up Docker network: acb_default_network
    2019/03/19 19:01:53 Setting up Docker configuration...
    2019/03/19 19:01:54 Successfully set up Docker configuration
    2019/03/19 19:01:54 Logging in to registry: mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Successfully logged into mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Executing step ID: acb_step_0. Working directory: '', Network: 'acb_default_network'
    2019/03/19 19:01:55 Launching container with name: acb_step_0
    
    Hello from Docker!
    This message shows that your installation appears to be working correctly.
    
    2019/03/19 19:01:56 Successfully executed container: acb_step_0
    2019/03/19 19:01:56 Step ID: acb_step_0 marked as successful (elapsed time in seconds: 0.843801)
    
    Run ID: cab was successful after 6s
    ```

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.
