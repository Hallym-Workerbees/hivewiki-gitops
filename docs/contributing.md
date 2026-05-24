# Contributing

이 문서는 HiveWiki GitOps 레포지토리에 변경을 추가할 때 지키는 기본 규칙을 정리합니다.

## Branch Names

작업 브랜치는 변경 성격이 드러나도록 만듭니다.

예시:

- `feat/prometheus-alertmanager-alerts`
- `fix/hivewiki-web-route`
- `docs/update-monitoring-alerts`

자동 생성 도구 이름이나 개인 이름은 브랜치명에 넣지 않습니다.

## Commit Messages

커밋 메시지는 영어로 작성하고 Conventional Commits 형식을 따릅니다.

예시:

```text
feat: add prometheus alertmanager alerts
fix: update hivewiki web service selector
docs: update monitoring alert documentation
```

자주 쓰는 type:

- `feat`: 새로운 기능이나 리소스 추가
- `fix`: 잘못된 설정 수정
- `docs`: 문서 변경
- `chore`: 동작 변경이 없는 관리 작업

## Secrets

Secret은 평문으로 커밋하지 않습니다. Kubernetes Secret이 필요한 경우 SealedSecret으로 변환해서
커밋합니다.

SealedSecret 파일명은 각 컴포넌트 디렉토리에서 `sealedsecret.yaml`을 사용합니다.

예시:

- `apps/hivewiki-web/dev/sealedsecret.yaml`
- `apps/hivewiki-collector/dev/sealedsecret.yaml`
- `infra/monitoring/secrets/resources/sealedsecret.yaml`

## Creating SealedSecrets

평문 Secret manifest를 임시 파일로 만든 뒤 `kubeseal`로 SealedSecret을 생성합니다. 임시 평문 파일은
커밋하지 않습니다.

Alertmanager Slack webhook 예시:

```bash
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  -f infra/monitoring/prometheus/slack-webhook-secret.yaml \
  -w infra/monitoring/secrets/resources/sealedsecret.yaml
```

이 예시에서 SealedSecret이 생성하는 Secret은 아래 값을 유지해야 합니다.

| Field | Value |
| --- | --- |
| Secret name | `alertmanager-slack-webhook` |
| Namespace | `monitoring` |
| Key | `webhook-url` |

Helm values나 Kubernetes manifest가 Secret을 참조할 때는 파일명이 아니라 생성되는 Secret의
`metadata.name`, namespace, key가 맞아야 합니다.
