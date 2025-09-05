---
lab:
  topic: Azure container services
  title: Azure CLI를 사용하여 Azure Container Apps에 컨테이너 배포하기
  description: Azure CLI 명령을 사용하여 안전한 Azure Container Apps 환경을 만들고 컨테이너를 배포하는 방법을 알아봅니다.
---

# Azure CLI를 사용하여 Azure Container Apps에 컨테이너 배포하기

이 연습에서는 Azure CLI를 사용하여 컨테이너화된 애플리케이션을 Azure Container Apps에 배포해 보세요. 이 연습을 통해 컨테이너 앱 환경을 생성하고, 컨테이너를 배포하며, 애플리케이션이 Azure에서 정상적으로 실행되는지 확인하는 방법을 학습하게 됩니다.

이 연습에서 수행된 작업:

* Azure에서 리소스 만들기
* Azure Container Apps 환경 만들기
* 환경에 컨테이너 앱 배포

이 연습을 완료하는 데 약 **15**분이 걸립니다.

## 리소스 그룹 만들기 및 Azure 환경 준비

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.

1. 페이지 상단의 검색 창 오른쪽에 있는 **[\>_]** 단추를 사용하여 Azure Portal에서 새 Cloud Shell을 만들고 ***Bash*** 환경을 선택합니다. Cloud Shell은 다음과 같이 Azure Portal 아래쪽 창에 명령줄 인터페이스를 제공합니다. 파일을 보관할 스토리지 계정을 선택하라는 프롬프트가 표시되면 **스토리지 계정 필요 없음**, 구독을 차례로 선택한 다음, **적용**을 선택합니다.

    > **참고**: 이전에 *PowerShell* 환경을 사용하는 Cloud Shell을 만든 경우 ***Bash***로 전환합니다.

1. 이 연습에 필요한 리소스에 대한 리소스 그룹을 만듭니다. **myResourceGroup**을 리소스 그룹에 사용하려는 이름으로 바꾸세요. 필요한 경우 **eastus**를 근처 지역으로 바꿀 수 있습니다. 사용하려는 리소스 그룹이 이미 있는 경우, 다음 단계를 진행하세요.

    ```azurecli
    az group create --location eastus --name myResourceGroup
    ```

1. 다음 명령을 실행하여 CLI용 Azure Container Apps 확장의 최신 버전이 설치되어 있는지 확인합니다.

    ```azurecli
    az extension add --name containerapp --upgrade
    ```

### 네임스페이스 등록

Azure Container Apps에 등록해야 하는 두 개의 네임스페이스가 있으며, 다음 단계에서 이러한 네임스페이스가 등록되어 있는지 확인합니다. 구독에 등록이 아직 구성되지 않은 경우, 각 등록을 완료하는 데 몇 분 정도 걸릴 수 있습니다. 

1. **Microsoft.App** 네임스페이스를 등록합니다. 

    ```bash
    az provider register --namespace Microsoft.App
    ```

1. 이전에 사용한 적이 없다면 Azure Monitor Log Analytics 작업 영역에 대한 **Microsoft.OperationalInsights** 공급자를 등록하세요.

    ```bash
    az provider register --namespace Microsoft.OperationalInsights
    ```

## Azure Container Apps 환경 만들기

Azure Container Apps의 환경은 컨테이너 앱 그룹 주위에 보안 경계를 만듭니다. 동일한 환경에 배포된 컨테이너 앱은 동일한 가상 네트워크에 배포되고 동일한 Log Analytics 작업 영역에 로그를 씁니다.

1. **az containerapp env create** 명령으로 환경을 만듭니다. **myResourceGroup** 및 **myLocation**을 이전에 사용한 값으로 바꾸세요. 작업을 완료하는 데 몇 분 정도 걸립니다.

    ```bash
    az containerapp env create \
        --name my-container-env \
        --resource-group myResourceGroup \
        --location myLocation
    ```

## 환경에 컨테이너 앱 배포

컨테이너 앱 환경 배포가 완료되면 컨테이너 이미지를 사용자 환경에 배포할 수 있습니다.

1. **containerapp create** 명령을 사용하여 샘플 앱 컨테이너 이미지를 배포합니다. **myResourceGroup**을 이전에 사용한 값으로 바꾸세요.

    ```bash
    az containerapp create \
        --name my-container-app \
        --resource-group myResourceGroup \
        --environment my-container-env \
        --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
        --target-port 80 \
        --ingress 'external' \
        --query properties.configuration.ingress.fqdn
    ```

    **--ingress**를 **external**로 설정하면 컨테이너 앱을 퍼블릭 요청에 사용할 수 있습니다. 명령이 앱에 액세스할 수 있는 링크를 반환합니다.

    ```
    Container app created. Access your app at <url>
    ```

배포를 확인하려면 **az containerapp create** 명령에서 반환된 URL을 선택하여 컨테이너 앱이 실행 중인지 확인하세요.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.
