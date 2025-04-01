# ArgoCD Notifications
![Image](https://github.com/user-attachments/assets/e70f0ad0-e919-445a-a876-f90643d85d33)

이 문서는 ArgoCD 에서 제공하는 Notifications 플러그인을 활용해, Slack 채널로 알림을 받기 위한 설정 방법을 설명합니다. Slack 알림을 기본으로 사용하며 별도의 `annotations`추가 없이 신규 애플리케이션에 알림이 적용되도록 하는 방법을 포함하고 있습니다. 


## 1. Overview
ArgoCD 는 Email, GitHub, Slack, Grafana, Webhook, Telegram, Teams 등의 다양한 알리 채널을 지원하며, ArgoCD 의 배포 결과를 모니터링 할 수 있도록 진행해야 하는데, 프로젝트 특성상 Slack 으로 알람 셋팅을 진행하도록 합니다.

#### 주요 기능:
- **배포 및 동기화 이벤트 알림**: `on-sync-succeeded`, `on-sync-failed`, `on-sync-running`, `on-sync-status-unknown`, `on-health-degraded`, `on-deployed` 등의 이벤트에 대해 알림 전송
- **기본 구독 설정**: ConfigMap의 `subscriptions`항목을 통해 신규 애플리케이션에도 별도의 `annotations`추가 없이 알림 적용
- **ApplicationSet 템플릿 지원**: ApplicationSet을 활용하여 자동으로 애플리케이션에 필요한 알림 `annotations`을 추가


## 2. 사전 요구사항
- **ArgoCD 설치 완료**: 클러스터에 ArgoCD가 설치되어 있어야 합니다.
- **Kubernetes Cluster**: Kubernetes 클러스터가 운영되고 있어야 합니다.
- **Slack API Token**: Slack API에서 토큰을 생성하고, 알림 전송에 사용할 채널(e.g. your-channel) 정보를 준비합니다.
- **kubectl**: Kubernetes 클러트서와 상호작용할 수 있는 CLI 툴


## 3. 설치 및 구성 단계

### 🔧 방법 1: CLI 기반 배포 (kubectl apply)
> kubectl apply 명령어로 각 `manifests`를 직접 배포하는 방식입니다.

#### 3-1. Notifications Controller 및 Catalog 설치
ArgoCD는 공식 확장 기능으로 `argocd-notifications`컨트롤러를 제공합니다. 아래 명령어를 통해 플러그인과 관련 catalog 리소스를 설치합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/release-1.0/manifests/install.yaml -n argocd
kubectl apply -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/release-1.0/catalog/install.yaml -n argocd
```

#### 3-2. Slack API Token 저장용 Secret 생성

`argocd-notifications`이 Slack에 접근할 수 있도록 Slack 토큰을 Secret에 저장

`argocd-notifications-secret.yaml`

```yaml
apiVersion: v1 
kind: Secret 
metadata: 
  name: argocd-notifications-secret 
  namespace: argocd
stringData:
  slack-token: "[SLACK_TOKEN]"  # Slack API Token으로 교체
```
> Secret 생성

```bash
kubectl apply -f argocd-notifications-secret.yaml -n argocd
```

#### 3-3. 알림 설정 ConfigMap 생성 또는 패치

알림 전송을 위한 서비스 설정과 기본 구독(subscription) 정보를 포함하는 ConfigMap을 생성 또는 패치합니다.
기본 구독을 설정하면, 신규 애플리케이션에도 별도 `annotations`없이 알림이 적용됩니다.

`argocd-notifications-cm.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
    username: Argo CD # optional
    icon: https://argo-cd.readthedocs.io/en/stable/assets/logo.png # optional
  subscriptions: |
    - recipients:
        - slack:<your-channel>
      triggers:
        - on-sync-status-unknown
        - on-sync-failed
        - on-health-degraded
        - on-deployed
        - on-sync-succeeded
```
> CM을 패치하여 적용

```bash
kubectl -n argocd patch cm argocd-notifications-cm --patch-file argocd-notifications-cm.yaml 
configmap/argocd-notifications-cm patched
```
> **주의 사항**:
> `subscriptions`항목에 기본 구독 정보가 포함되어 있더라도, 일부 버전에서는 애플리케이션 메타데이터에 개별 `annotations`을 추가해야 알림이 정상적으로 동작할 수 있습니다.

#### 3-4. Slack 채널 및 애플리케이션 설정

알림 컨트롤러는 Slack 토큰만 보유하고 있으므로, 실제 알림 전송 시 대상 Slack 채널 정보가 필요합니다. 때문에 어떤 애플리케이션을 구독할 것인지 정해줘야 합니다.
이를 위해 두 가지 방법을 사용할 수 있습니다.

**A. 개별 애플리케이션에 `annotations`추가**

```bash
export APP_NAME=test-app
export NAMESPACE=argocd
export SLACK_CHANNEL_NAME=your-channel # '#' 없이 채널 이름만 사용

# Sync 성공시 알림
kubectl patch app $APP_NAME -n $NAMESPACE -p '{"metadata": {"annotations": {"notifications.argoproj.io/subscribe.on-sync-succeeded.slack":"$SLACK_CHANNEL_NAME"}}}' --type merge
 
# Sync 실패시 알림
kubectl patch app $APP_NAME -n $NAMESPACE -p '{"metadata": {"annotations": {"notifications.argoproj.io/subscribe.on-sync-failed.slack":"$SLACK_CHANNEL_NAME"}}}' --type merge
 
# Sync 진행중 알림
kubectl patch app $APP_NAME -n $NAMESPACE -p '{"metadata": {"annotations": {"notifications.argoproj.io/subscribe.on-sync-running.slack":"$SLACK_CHANNEL_NAME"}}}' --type merge
 
# Sync 상태가 Unknown일 때
kubectl patch app $APP_NAME -n $NAMESPACE -p '{"metadata": {"annotations": {"notifications.argoproj.io/subscribe.on-sync-status-unknown.slack":"$SLACK_CHANNEL_NAME"}}}' --type merge
 
# Health가 degrade 되었을 때
kubectl patch app $APP_NAME -n $NAMESPACE -p '{"metadata": {"annotations": {"notifications.argoproj.io/subscribe.on-health-degraded.slack":"$SLACK_CHANNEL_NAME"}}}' --type merge
 
# deploy 되었을 때
kubectl patch app $APP_NAME -n $NAMESPACE -p '{"metadata": {"annotations": {"notifications.argoproj.io/subscribe.on-deployed.slack":"$SLACK_CHANNEL_NAME"}}}' --type merge
```
> **주의사항:**
> - Slack 채널 이름은 `#your-channel`이 아닌 `your-channel`과 같이 입력합니다.
> - 해당 Slack 채널에 `argocd-notifications`봇이 초대되어 있어야 합니다.

**B. ApplicationSet 템플릿에 기본 `annotations`추가**

ApplicationSet을 통해 애플리케이션을 생성하는 경우, 템플릿에 기본 어노테이션을 포함시켜 신규 애플리케이션에 자동으로 알림 설정을 적용할 수 있습니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-app
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: "your-channel"
    notifications.argoproj.io/subscribe.on-sync-failed.slack: "your-channel"
    notifications.argoproj.io/subscribe.on-sync-running.slack: "your-channel"
    notifications.argoproj.io/subscribe.on-sync-status-unknown.slack: "your-channel"
    notifications.argocd.argoproj.io/subscribe.on-health-degraded.slack: "your-channel"
    notifications.argocd.argoproj.io/subscribe.on-deployed.slack: "your-channel"
spec:
  # 애플리케이션 스펙 정의
```
> 이 방법을 사용하면, 매번 수동으로 `annotations`을 추가하지 않아도 기본 `annotations`이 자동으로 적용됩니다.

### 📦 방법 2: Helm Chart 기반 배포
> 작성한 Helm Chart를 통해 ArgoCD Notifications를 패키지화하여 GitOps 방식으로 관리하려는 경우입니다.

#### 3-1. `values.yaml` 준비
`values-argocd-notification.yaml`
```yaml
image:
  repository: argoprojlabs/argocd-notifications
  tag: v1.2.0

notifiers:
  service.slack: |
    token: $slack-token
    username: ArgoCD  # optional
    icon: https://argo-cd.readthedocs.io/en/stable/assets/logo.png # optional

subscriptions:
  - recipients:
    - ""
    triggers:
      - on-sync-status-unknown
      - on-sync-failed
      - on-health-degraded
      - on-deployed
      - on-sync-succeeded

argocdUrl: <argocdUrl>
```

#### 3-2. ArgoCD Helm Chart Repository 추가 및 argocd-notifications APP 생성
CI Repo 등록
- Argocd Web UI → Settings → Repositores → + CONNECT REPO → 등록 진행

argocd-notifications APP 생성
- Argocd Web UI → Applications → + NEW APP → APP 생성 진행(Helm Parameter 에서 secret 에 Slack API Token, Subscriptions 에 Slack Channel 입력)

### 📂 방법 3: Kustomize 기반 배포
> GitOps 방식 또는 Kustomize로 ArgoCD Notification 리소스를 관리하는 경우입니다.

#### 3-1. `kustomizations.yaml` 배포
argocd-notifications APP 생성
- Argocd Web UI → Applications → + NEW APP → APP 생성 / controller 디렉토리의 `kustomizations.yaml`로 배포 진행

#### 3-2. secret Slack Token 주입
`argocd-notifications-secret.yaml` 파일에서 Slack API 토큰을 `slack-token` 키로 주입합니다.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
stringData:
  slack-token: "<SLACK_TOKEN>"  # 실제 Slack API Token으로 교체
```

#### 3-3. cm subscriptions 알림 구독 기본 설정
`argocd-notifications-cm.yaml`
```yaml
data:
  service.slack: |
    token: $slack-token
    username: Argo CD
    icon: https://argo-cd.readthedocs.io/en/stable/assets/logo.png
  subscriptions: |
    - recipients:
        - slack:<your-channel>
      triggers:
        - on-sync-status-unknown
        - on-sync-failed
        - on-health-degraded
        - on-deployed
        - on-sync-succeeded
```

## 4. Slack 채널 알림 설정 (공통)
> CLI 든 Helm 이든 Kustomize 든, 모두 동일하게 적용 됩니다.
- 개별 애플리케이션에 `annotations`추가
- ApplicationSet 템플릿에 `annotations`포함

## 5. 문제 해결 및 주의사항
**subscriptions(기본 구독) 설정 미적용 문제:**
- `argocd-notifications-cm`의 `subscriptions`항목이 올바르게 구성되지 않았거나 누락되면 알림이 전송되지 않을 수 있습니다.
- 이 경우, 애플리케이션 `matadata`에 개별 `annotations`을 추가하여여 정상 동작 여부를 확인 또는 Helm Chart로 배포한 경우 `subscriptions`의 `trigger`필드에 값이 정상적으로 들어가 있는지 확인

![Image](https://github.com/user-attachments/assets/869c2f5f-ee44-41a1-974c-42cd956e85a2)

**버전 관련 이슈:**
- 일부 버전에서 `--secret-name`등의 `command`옵션이 동작하지 않는 이슈 발생 때문에 argocd-notification v1.2.0 으로 진행

**Helm Chart를 사용하는 경우**
- Helm Chart를 통한 배포 시 override values 파일(`values-argocd-notification.yaml`)을 사용하여 Slack 토큰과 채널 정보를 주입할 수 있고, ArgoCD 배포시 Parameter 값에 주어 주입 할 수 있습니다.
- 배포 후, 알림 동작 여부를 확인하고 필요 시 애플리케이션 어노테이션 설정을 재검토하세요.

## 5. 참고 자료
- ArgoCD Notifications 공식 문서
https://argo-cd.readthedocs.io/en/latest/operator-manual/notifications/
- ArgoCD GitHub Repository 및 관련 이슈 트래커

## 6. 구성 완료(alert check)
![Image](https://github.com/user-attachments/assets/9586ff13-e17e-4fc2-ae4e-e411c71d69a7)