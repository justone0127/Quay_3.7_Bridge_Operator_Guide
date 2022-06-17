## Quay 3.7 Bridge Operator Guide

Quay Bridge Operator를 사용하여 OpenShift의 통합 컨테이너 레스트리를 Red Hat Quay 레지스트리로 교체할 수 있습니다. 이를 통해 통합 OpenShift 레지스트리는 RBAC(Role Baced Access Control) 기능이 향상된 고 가용성 엔터프라이즈급 Red Hat Quay 레지스트리가 됩니다.



Bridge Operator의 주요 목표는 통합 OpenShift 레지스트리의 기능을 새로운 Red Hat Quay 레지스트리에 복제하는 것 입니다. 이 Operator로 활성화된 기능은 다음과 같습니다.


- OpenShift 네임스페이스를 Red Hat Quay 조직으로 동기화합니다.

  - 각 기본 네임스페이스 서비스 계정에 대한 로봇 계정 생성
  - 생성된 각 로봇 계정에 대한 암호 생성 (각 로봇 암호를 서비스 계정에 탑재 가능 및 이미지 풀 암호로 연결)
  - OpenShift ImageStreams를 Quay 저장소로 동기화
- ImageStreams를 사용하여 Red Hat Quay로 출력하는 새 빌드를 자동으로 다시 작성
- 빌드가 완료되면 ImageStream 태그 자동 가져오기

![15_quay_bridge_operator](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/15_quay_bridge_operator.png)



### 1. Quay and OpenShift Entity Mapping

| Quay Entity                                 | OpenShift Entity                           |
| ------------------------------------------- | ------------------------------------------ |
| Organization                                | Project/Namespace                          |
| Repository                                  | ImageStream                                |
| Robot Accounts                              | ServiceAccount                             |
| Quay Team                                   | Group                                      |
| Build (docker build trigger by git actions) | Build (s2i, docker build, jenkins, tekton) |



### 2. Prerequisites

- `superuser` 권한이 있는 Red Hat Quay 환경
- `cluster-admin` 권한이 있는 Red Hat OpenShift Container Platform 환경 (4.2 이상 권장)
- OpenShift command line 도구 (`oc` command)

**Quay 3.6 버전부터는 Quay Bridge Operator(QBO)를 설치하면 더 이상 Webhook 인증서를 수동으로  생성하는 스크립트를 실행할 필요가 없습니다. Operator 설치 시 MutatingWebhookConfiguration이 자동으로 생성됩니다.**

### 3. Setting up and configuring OpenShift and Red Hat Quay

- Red Hat Quay 및 OpenShift 구성이 모두 필요합니다.

### 3.1 Red Hat Quay Setup

전용 Red Hat Quay 조직을 생성하고 해당 조직 내에서 생성하는 새 애플리케이션에서 OpenShift의 Quay Bridge Operator와 함께 사용할 OAuth 토큰을 생성합니다.

- 조직 생성

  Quay Console 접속 > Create New Organization 선택 > 조직 생성 > `test` 조직 생성

  ![01_new_organization](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/01_new_organization.png)

- Application 생성

  Application 메뉴 선택 > `Create New Application` 버튼을 선택하고 Application의 이름을 입력한 후 Application 생성

  - 가이드에서는 `openshift`라는 새로운 Application을 생성함

    ![02_new_application](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/02_new_application.png)

  - 생성된 `openshift` Application 목록 확인

    ![03_new_application02](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/03_new_application02.png)

- Token 생성

  생성된 `openshift` 선택 > 왼쪽 탐색 메뉴에서 **Generate Token** 메뉴 선택 > 아래 권한 설정 선택 > **Generate Access Token** 버튼을 눌러서 Token 생성

  ![04_generate_token](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/04_generate_token.png)

- 생성할 토큰에 대한 권한을 검토한 후에 **Authorize Application** 버튼을 선택하여 토큰 생성을 완료

  ![05_authorize_application](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/05_authorize_application.png)

- 생성된 액세스 Token은 OpenShift에 등록해야 하므로 복사해 두어야 함

  ![06_access_token](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/06_access_token.png)

### 3.2 OpenShift Setup

Quay Bridge Operator에 대해 OpenShift를 설정하려면 다음을 포함한 여러 단계가 필요합니다.

### 3.2.1 Operator 배포

OpertorHub 선택 > Quay Bridge Operator 검색 > Install > all namespaces Mode, Update Channel, Approval Strategy 선택 > Subscribe 선택

![07_quay_bridge_operator](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/07_quay_bridge_operator.png)

### 3.2.2 OAuth Token에 대한 OpenShift Secret 생성

- `oc` command를 사용하여 `cluster-admin` 계정으로 OpenShift에 로그인

- `openshift-operators` 네임스페이스를 선택하거나 새로운 네임스페이스를 생성

- OpenShift secret을 생성할 때 <access_token>은 Red Hat Quay에서 생성한 Token으로 대체합니다. 예를 들어, 다음은 `token`이라는 key로 `quay-integration`이라는 <access_token>을 사용하여 secret을 생성합니다.

  ```bash
  $ oc create secret generic quay-integration --from-literal=token=<access_token> -n openshift-operators
  ```


### 3.2.3 QuayIntegration 사용자 지정 리소스 생성

OpenShift와 Quay간의 통합을 완료하려면 `QuayIntegration` 사용자 지정 리소스를 만들어야 합니다. 이것은 웹 콘솔이나 명령어를 통해 생성할 수 있습니다.

- quay-integration.yaml

  ```yaml
  apiVersion: quay.redhat.com/v1
  kind: QuayIntegration
  metadata:
    name: example-quayintegration
  spec:
    clusterID: openshift  1
    credentialsSecret:
      namespace: openshift-operators 
      name: quay-integration 2
    quayHostname: https://<QUAY_URL>   3
    insecureRegistry: false  4
  ```

> 1 : ClusterID 값은 전체 ecosystem에서 고유해야 합니다. 이 값은 선택사항이며 기본값은 openshift입니다.
>
> 2: credentialSecret은 이전에 생성된 Token을 포함하는 Secret 이름과 네임스페이스를 참조합니다.
>
> 3: <QUAY_URL>을 구성된 Red Hat Quay 인스턴스로 대체
>
> 4: Quay가 자체 서명된 인증서를 사용하는 경우 `insecureRegistry : true`로 설정합니다.

- 사용자 지정 리소스 생성

  ```bash
  $ oc create -f quay-integration.yaml
  ```



### 4. Quay Registry Insecure 설정 추가

자체 서명된 인증서의 경우 x509 에러가 발생하므로, 이를 방지하기 위해 `image.config.openshift.io/cluster` 리소스를 편집하여 이미지 레지스트리 구성을 변경합니다.

- OCP 콘솔 > Search > Image (Config) > Cluster > YAML 선택 > spec 아래 부분에 내용 추가

  ```bash
  spec:
    allowedRegistriesForImport:
      - domainName: ${QUAY_HOST}
        insecure: true
    registrySources:
      insecureRegistries:
        - ${QUAY_HOST}
  ```



### 5. Application Deployment

OpenShift에서 프로젝트를 생성하여 Application을 배포하면, 해당 프로젝트에서 빌드된 이미지는 OpenShift의 내부 레지스트리가 아닌 Quay에 Push가 됩니다. ClusterID(openshift) 접두사가 붙은 프로젝트가 조직으로 생성되고 해당 조직 내에 애플리케이션 이름으로 이미지 저장소가 생성되며 해당 이미지 저장소에 이미지가 Push 됩니다.

### 5.1 Project 생성

### 5.1.1 새로운 프로젝트 생성

OpenShift Console 또는 CLI 명령을 통해 프로젝트를 생성합니다.

![09_new_project](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/09_new_project.png)

```bash
$ oc new-project quay-demo
```

### 5.1.2 샘플 애플리케이션 배포

개발자 콘솔 이동 > +Add > Developer Catalog : All services > Basic Spring Boot선택 > Create Application

![10_sample_app](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/10_sample_app.png)

![11_create_app](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/11_create_app.png)

>  애플리케이션이 배포 된 후에 해당 이미지는 OpenShift의 기본 이미지 레지스트리가 아닌 Quay에 Push된 것을 확인 할 수 있습니다.

### 5.1.3 Qauy 콘솔 이동

openshift_quay_demo 조직이 새로 생성 되었고,  해당 조직 아래에 `devfile-sample-java-springboot-basic-git` repository가 새로 생성된 것을 확인 할 수 있습니다.

![12_quay_console](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/12_quay_console.png)

> 또한,  User and Organizations 부분을 확인해보면, 해당 클러스터 내의 openshift 접두사가 붙지 않은 프로젝트에 대한 모든 조직이 생성된 것을 볼 수 있습니다.

### 5.1.4 Repository 확인

생성된 Repository 안에 Push된 이미지가 확인되며, 해당 이미지에 대한 보안 취약점 결과도 확인 가능합니다. 

![13_quay_repository](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/13_quay_repository.png)



### 5.1.5 보안 취약점 스캐닝 결과

Security Scan 결과를 선택하면, 다음과 같이 이미지에 대한 보안 취약점 결과도 상세하게 확인 가능합니다.

![14_security_scanning](https://github.com/justone0127/Quay_3.7_Bridge_Operator_Guide/blob/main/images/14_security_scanning.png)







참고 URL)

https://access.redhat.com/documentation/ko-kr/red_hat_quay/3.7/html/manage_red_hat_quay/quay-bridge-operator (Integrate Red Hat Quay into OpenShift with the Bridge Operator)

https://docs.openshift.com/container-platform/4.10/openshift_images/image-configuration.html (Image Configuration)

https://github.com/quay/quay-bridge-operator

