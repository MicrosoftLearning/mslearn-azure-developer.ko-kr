---
lab:
  topic: Azure container services
  title: Azure CLI 명령을 사용하여 Azure Container Instances에 컨테이너 배포하기
  description: Azure CLI 명령을 사용하여 컨테이너를 Azure Container Instances에 배포하는 방법을 알아보세요.
---

# Azure CLI 명령을 사용하여 Azure Container Instances에 컨테이너 배포하기

이 연습에서는 Azure CLI를 사용하여 ACI(Azure Container Instances)에 컨테이너를 배포하고 실행합니다. 이 연습을 통해 컨테이너 그룹을 생성하고, 컨테이너 설정을 지정하며, 클라우드에서 컨테이너화된 애플리케이션이 정상적으로 실행되는지 확인하는 방법을 학습하게 됩니다.

이 연습에서 수행된 작업:

* Azure에서 Azure Container Instance 리소스 만들기
* 컨테이너 만들기 및 배포
* 컨테이너가 실행 중인지 확인

이 연습을 완료하는 데 약 **15**분이 걸립니다.

## 리소스 그룹 만들기

1. 브라우저에서 Azure Portal[https://portal.azure.com](https://portal.azure.com)로 이동한 다음, 메시지가 나타나면 Azure 자격 증명을 사용하여 로그인합니다.

1. 페이지 상단의 검색 창 오른쪽에 있는 **[\>_]** 단추를 사용하여 Azure Portal에서 새 Cloud Shell을 만들고 ***Bash*** 환경을 선택합니다. Cloud Shell은 다음과 같이 Azure Portal 아래쪽 창에 명령줄 인터페이스를 제공합니다. 파일을 보관할 스토리지 계정을 선택하라는 프롬프트가 표시되면 **스토리지 계정 필요 없음**, 구독을 차례로 선택한 다음, **적용**을 선택합니다.

    > **참고**: 이전에 *PowerShell* 환경을 사용하는 Cloud Shell을 만든 경우 ***Bash***로 전환합니다.

1. 이 연습에 필요한 리소스에 대한 리소스 그룹을 만듭니다. **myResourceGroup**을 리소스 그룹에 사용하려는 이름으로 바꾸세요. 필요한 경우 **eastus**를 근처 지역으로 바꿀 수 있습니다. 사용하려는 리소스 그룹이 이미 있는 경우, 다음 단계를 진행하세요.

    ```
    az group create --location eastus --name myResourceGroup
    ```

## 컨테이너 만들기 및 배포

**az container create** 명령에 이름, Docker 이미지 및 Azure 리소스 그룹을 제공하여 컨테이너를 만듭니다. DNS 이름 레이블을 지정하여 컨테이너를 인터넷에 노출할 것입니다.

1. 다음 명령을 실행하여 컨테이너를 인터넷에 노출하는 데 사용되는 DNS 이름을 만듭니다. DNS 이름은 고유해야 합니다. Cloud Shell에서 이 명령을 실행하여 고유한 이름을 저장하는 변수를 만듭니다.

    ```bash
    DNS_NAME_LABEL=aci-example-$RANDOM
    ```

1. 다음 명령을 실행하여 컨테이너 인스턴스를 만듭니다. **myResourceGroup** 및 **myLocation**을 이전에 사용한 값으로 바꾸세요. 작업을 완료하는 데 몇 분 정도 걸립니다.

    ```bash
    az container create --resource-group myResourceGroup \
        --name mycontainer \
        --image mcr.microsoft.com/azuredocs/aci-helloworld \
        --ports 80 \
        --dns-name-label $DNS_NAME_LABEL --location myLocation \
        --os-type Linux \
        --cpu 1 \
        --memory 1.5 
    ```

    이전 명령에서 **$DNS_NAME_LABEL**은 DNS 이름을 지정합니다. 이미지 이름 **mcr.microsoft.com/azuredocs/aci-helloworld**는 기본 Node.js 웹 애플리케이션을 실행하는 Docker 이미지를 참조합니다.

**az container create** 명령이 완료되면 다음 섹션으로 이동하세요.

## 컨테이너가 실행 중인지 확인

**az container show** 명령을 사용하여 컨테이너 빌드 상태를 확인할 수 있습니다. 

1. 다음 명령을 실행하여 만든 컨테이너의 프로비전 상태를 확인합니다. **myResourceGroup**을 이전에 사용한 값으로 바꾸세요.

    ```bash
    az container show --resource-group myResourceGroup \
        --name mycontainer \
        --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
        --out table 
    ```

    컨테이너의 FQDN(정규화된 도메인 이름) 및 프로비전 상태를 확인합니다. 예를 들어 다음과 같습니다.

    ```
    FQDN                                    ProvisioningState
    --------------------------------------  -------------------
    aci-wt.eastus.azurecontainer.io         Succeeded
    ```

    > **참고:** 컨테이너가 **생성 중** 상태일 경우 **성공** 상태가 표시될 때까지 잠시 기다렸다가 명령을 다시 실행합니다.

1. 브라우저에서 컨테이너의 FQDN으로 이동하여 실행 중인지 확인합니다. 사이트가 안전하지 않다는 경고가 표시될 수 있습니다.

## 리소스 정리

연습을 마쳤으므로 불필요한 리소스 사용을 방지하기 위해 만든 클라우드 리소스를 삭제해야 합니다.

1. 만든 리소스 그룹으로 이동하여 이 연습에 사용된 리소스의 내용을 봅니다.
1. 도구 모음에서 **리소스 그룹 삭제**를 선택합니다.
1. 리소스 그룹 이름을 입력하고 삭제할 것인지 확인합니다.

> **주의:** 리소스 그룹을 삭제하면 해당 리소스 그룹에 포함된 모든 리소스가 함께 삭제됩니다. 이 연습을 위해 기존 리소스 그룹을 선택한 경우, 이 연습의 범위와 관계없는 기존 리소스도 삭제됩니다.
