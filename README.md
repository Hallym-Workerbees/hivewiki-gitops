# HiveWiki GitOps

HiveWiki GitOps 레포지토리는 HiveWiki 프로젝트의 Kubernetes 배포 구성을 관리합니다.
ArgoCD를 사용해 애플리케이션, 공통 인프라, 모니터링 구성을 GitOps 방식으로 동기화합니다.

## Repository Layout

```text
apps/                 # HiveWiki 애플리케이션 Kubernetes manifest
argocd/               # ArgoCD root/app-of-apps 및 Application manifest
infra/                # 공통 인프라 Helm values 및 manifest
docs/                 # 운영 문서
```

주요 흐름은 `argocd/hivewiki-root.yaml`에서 시작합니다. Root Application이 `argocd/apps`와
`argocd/infra` 아래의 Application manifest를 재귀적으로 동기화하고, 각 Application이 실제
`apps/` 또는 `infra/` 경로를 바라봅니다.

## Applications

현재 애플리케이션 환경은 `dev` 기준으로 관리됩니다.

| Application | Namespace | Path |
| --- | --- | --- |
| `hivewiki-web-dev` | `hivewiki-web-dev` | `apps/hivewiki-web/dev` |
| `hivewiki-collector-dev` | `hivewiki-collector-dev` | `apps/hivewiki-collector/dev` |

## Infrastructure

공통 인프라는 `argocd/infra`의 ArgoCD Application과 `infra/`의 Helm values 또는 manifest로 관리합니다.

| Component | Namespace | Main config |
| --- | --- | --- |
| Prometheus / Alertmanager | `monitoring` | `infra/monitoring/prometheus/values.yaml` |
| Grafana | `monitoring` | `infra/monitoring/grafana/values.yaml` |
| YACE | `monitoring` | `infra/monitoring/yace/values.yaml` |
| Loki / Alloy | `monitoring` | `infra/monitoring/loki`, `infra/monitoring/alloy` |
| Gateway API | cluster / gateway namespace resources | `infra/gateway-api` |
| ExternalDNS | `external-dns` | `infra/external-dns/values.yaml` |
| KEDA | `keda` | `infra/keda/values.yaml` |
| kube-green | `kube-green` | `infra/kube-green` |
| Sealed Secrets | `kube-system` | `infra/sealed-secrets/values.yaml` |

## Documentation

- [Contributing](docs/contributing.md): 브랜치, 커밋 메시지, Secret 관리 규칙
- [Monitoring alerts](docs/monitoring-alerts.md): Prometheus Alertmanager 연동과 알림 기준

## Local Setup

필수 도구:

- Kubernetes CLI
- ArgoCD CLI
- Helm
- pre-commit
- gitleaks
- commitizen

`pre-commit`은 Python 기반 도구입니다.

```bash
pip install pre-commit
pre-commit install --hook-type pre-commit --hook-type commit-msg
```

커밋 메시지는 영어로 작성하고 Conventional Commits 규칙을 따릅니다.
자세한 규칙은 [Commitizen 문서](https://commitizen-tools.github.io/commitizen/tutorials/writing_commits/)를 참고하세요.
