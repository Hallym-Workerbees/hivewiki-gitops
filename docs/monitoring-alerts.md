# Monitoring Alerts

Prometheus와 Alertmanager는 `prometheus-community/prometheus` Helm chart로 배포합니다.
`kube-prometheus-stack`은 사용하지 않습니다.

주요 설정 파일:

- ArgoCD Application: `argocd/infra/prometheus.yaml`
- Helm values: `infra/monitoring/prometheus/values.yaml`
- Slack webhook SealedSecret: `infra/monitoring/secrets/resources/sealedsecret.yaml`
- Grafana Loki alert rules: `infra/monitoring/grafana/values.yaml`
- Grafana dashboards: `infra/monitoring/grafana/dashboards/`

## Collector Metrics Scrape

HiveWiki collector는 `/metrics` endpoint를 노출하며, Prometheus annotation 기반으로 scrape합니다.

collector dev 환경의 scrape 설정:

- Workload namespace: `hivewiki-collector-dev`
- Service: `apps/hivewiki-collector/dev/service.yaml`
- Metrics path: `/metrics`
- Metrics port: `8080`
- Prometheus job: `hivewiki-collector`

collector Deployment pod template과 Service에는 아래 annotation을 둡니다.

```yaml
prometheus.io/scrape: "true"
prometheus.io/path: /metrics
prometheus.io/port: "8080"
```

Prometheus Helm values의 `extraScrapeConfigs`에는 `role: endpoints` discovery를 사용하는
`hivewiki-collector` job이 있습니다. 이 job은 `prometheus.io/scrape: "true"` annotation이 붙은
`hivewiki-collector-dev/hivewiki-collector` Service만 keep하고, annotation의 path와 port를 scrape 주소로
반영합니다.

Grafana dashboard는 `infra/monitoring/grafana/dashboards/hivewiki-collector-dev.json`으로 관리합니다.
기존 Grafana sidecar provisioning과 동일하게 `grafana_dashboard: "1"` label이 붙은 ConfigMap으로 생성됩니다.

## Alertmanager Integration

Prometheus는 Alertmanager service를 직접 바라봅니다.

```yaml
server:
  alertmanagers:
    - static_configs:
        - targets:
            - prometheus-alertmanager.monitoring.svc.cluster.local:9093
```

Slack webhook URL은 values에 평문으로 넣지 않습니다. Alertmanager는 Secret mount의 파일을 읽습니다.

```yaml
api_url_file: /etc/alertmanager/secrets/slack/webhook-url
```

이 파일은 `alertmanager-slack-webhook` Secret의 `webhook-url` key에서 옵니다.

## Routing Policy

Alertmanager route 정책:

| Setting | Value |
| --- | --- |
| `group_by` | `alertname`, `service`, `env`, `severity` |
| `group_wait` | `30s` |
| `group_interval` | `5m` |
| default `repeat_interval` | `4h` |
| critical `repeat_interval` | `1h` |

억제 정책:

- 같은 `service`와 `env`에서 `critical` alert가 발생하면 `warning`과 `info` alert를 억제합니다.
- `equal` 기준은 `service`, `env`입니다.

## Alert Labels

Alert rule은 가능한 한 아래 labels를 포함합니다.

- `severity`
- `service`
- `env`

현재 `env`는 `dev`로 설정되어 있습니다.

## Environment Strategy

현재 활성 alert rule은 `dev` 환경 기준입니다. 아직 `prod` 워크로드가 없으므로 prod alert rule은 만들지 않습니다.
대신 모든 alert에 `env` label을 유지해서 나중에 prod가 추가될 때 routing과 grouping을 분리할 수 있게 합니다.

prod 환경을 추가할 때는 아래 항목을 먼저 확인합니다.

- prod namespace 이름
- HiveWiki web app의 prod Prometheus job 또는 namespace label
- YACE metric에 AWS `Environment` tag가 어떤 Prometheus label로 들어오는지
- kube-green 로그에서 `sleepinfo.namespace`로 dev/prod를 안정적으로 구분할 수 있는지
- dev/prod Slack contact point 또는 Slack channel을 분리할지

운영 정책은 prod를 더 강하게, dev를 더 느슨하게 가져갑니다.

| Environment | Policy |
| --- | --- |
| `dev` | 필요한 알림만 활성화하고, notification policy의 `repeat_interval`을 길게 둡니다. |
| `prod` | 장애성 알림을 활성화하고, critical alert의 반복 주기를 dev보다 짧게 둡니다. |

prod alert를 추가할 때는 기존 dev rule을 단순 복사하지 말고 namespace, job, AWS tag label, Loki log pattern이
prod 리소스만 바라보는지 확인한 뒤 `env: prod` label을 부여합니다.

## Alert Rules

| Alert | Severity | Service | Duration | Condition |
| --- | --- | --- | --- | --- |
| `ArgoCDAppUnhealthy` | warning | `argocd` | `5m` | `argocd_app_health_status{health_status!="Healthy"} == 1` |
| `ArgoCDAppOutOfSync` | warning | `argocd` | `10m` | `argocd_app_sync_status{sync_status!="Synced"} == 1` |
| `PodCrashLooping` | warning | `kubernetes` | `5m` | `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} > 0` |
| `PodOOMKilled` | warning | `kubernetes` | `5m` | `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0` |
| `PrometheusTargetDown` | critical | `monitoring` | `5m` | `up == 0` |
| `NodeNotReady` | critical | `kubernetes` | `5m` | `kube_node_status_condition{condition="Ready", status="true"} == 0` |
| `PersistentVolumeUsageHigh` | warning | `kubernetes` | `10m` | `kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes >= 0.85` |
| `ALB5xxHigh` | warning | `alb` | `5m` | ALB ELB/Target 5xx count divided by request count is over `1%`, and request count is over `100` per CloudWatch period |
| `ALBTargetResponseTimeP95High` | warning | `alb` | `5m` | `aws_applicationelb_target_response_time_p95 > 0.5` |
| `RDSCPUHigh` | warning | `rds` | `10m` | `aws_rds_cpuutilization_average > 80` |
| `RDSReadLatencyHigh` | warning | `rds` | `10m` | `aws_rds_read_latency_average > 0.05` |
| `RDSWriteLatencyHigh` | warning | `rds` | `10m` | `aws_rds_write_latency_average > 0.05` |
| `WebAppDown` | critical | `hivewiki-web` | `3m` | `hivewiki_up{job="hivewiki-web"} == 0` |
| `WebAppHigh5xxRate` | warning | `hivewiki-web` | `5m` | HiveWiki 5xx response rate is over `1%`, and response rate is over `1 rps` |
| `WebAppHighLatencyP95` | warning | `hivewiki-web` | `5m` | `histogram_quantile(0.95, rate(hivewiki_http_request_duration_seconds_bucket[5m])) > 0.5` |
| `CollectorScrapeDown` | critical | `hivewiki-collector` | `3m` | `up{job="hivewiki-collector"} == 0` |
| `CollectorNoRecentSuccess` | warning | `hivewiki-collector` | `5m` | `time() - collector_last_success_timestamp{job="hivewiki-collector"} > 1800` |
| `CollectorConsecutiveFailures` | critical | `hivewiki-collector` | `5m` | `collector_consecutive_failures{job="hivewiki-collector"} >= 3` |
| `CollectorDispatchFailures` | warning | `hivewiki-collector` | `5m` | `increase(collector_dispatch_failed_total{job="hivewiki-collector"}[10m]) > 0` |
| `CollectorLoopErrors` | warning | `hivewiki-collector` | `5m` | `increase(collector_loop_errors_total{job="hivewiki-collector"}[10m]) > 0` |
| `CollectorPendingDocumentsHigh` | warning | `hivewiki-collector` | `15m` | `collector_pending_documents{job="hivewiki-collector"} > 100` |
| `CollectorDeadDocumentsPresent` | warning | `hivewiki-collector` | `10m` | `collector_dead_documents{job="hivewiki-collector"} > 0` |

## 제외한 알림

`kube-green` 알림은 Prometheus rule에서 제외합니다. 현재 kube-green은 필요한 metric을 노출하지 않으므로
Loki 로그 기반 alert로 별도 처리합니다.

`KarpenterUnschedulablePods`는 정의하지 않습니다. 현재 사용 가능한 Karpenter metric에는
`karpenter_scheduler_unschedulable_pods_count`처럼 unschedulable pod 수를 직접 나타내는 metric이 없습니다.

`RDSFreeStorageLow`는 정의하지 않습니다. 현재 RDS metric에는 남은 스토리지 용량만 있고 전체 또는 할당
스토리지 용량이 없어 Prometheus metric만으로 안전한 잔여 비율을 계산할 수 없습니다.

## Grafana Loki 알림

kube-green은 Prometheus metric을 노출하지 않으므로 sleep/wake 이벤트는 Loki 로그 기반 Grafana Alerting으로
감지합니다. Grafana alert rule provisioning은 Grafana Helm chart의 `alerting` values로 관리합니다.
이 값은 chart에 의해 Grafana의 alerting provisioning 디렉토리인 `/etc/grafana/provisioning/alerting`에
반영됩니다.

현재 Loki datasource uid는 `loki`를 사용합니다.

| Alert | Severity | Service | Duration | Condition |
| --- | --- | --- | --- | --- |
| `KubeGreenWakeUp` | info | `kube-green` | `0m` | `controllers.SleepInfo`, `last schedule value`, `"operation":"WAKE_UP"` 로그가 최근 `5m` 안에 1건 이상 |
| `KubeGreenSleep` | info | `kube-green` | `0m` | `controllers.SleepInfo`, `last schedule value`, `"operation":"SLEEP"` 로그가 최근 `5m` 안에 1건 이상 |

사용하는 LogQL 패턴:

```logql
sum(count_over_time({namespace="kube-green"} |= "controllers.SleepInfo" |= "last schedule value" |= "\"operation\":\"WAKE_UP\"" [5m]))
sum(count_over_time({namespace="kube-green"} |= "controllers.SleepInfo" |= "last schedule value" |= "\"operation\":\"SLEEP\"" [5m]))
```

로그 검색 범위를 `namespace="kube-green"`, `controllers.SleepInfo`, `last schedule value`, operation 값으로
좁혀 sleep/wake 실행과 직접 관련된 로그만 감지합니다. rule group interval은 `1m`, `noDataState`는 `OK`,
`execErrState`는 `Error`입니다.

Slack contact point와 notification policy는 별도로 구성합니다. Slack webhook URL은 provisioning 파일에
평문으로 넣지 않고 Prometheus Alertmanager에서도 사용하는 `alertmanager-slack-webhook` Secret을 Grafana pod
환경변수로 주입한 뒤 `$GRAFANA_ALERTING_SLACK_WEBHOOK_URL`로 참조합니다. Grafana alerting file provisioning은
환경변수 치환을 지원합니다.

Grafana values:

```yaml
envValueFrom:
  GRAFANA_ALERTING_SLACK_WEBHOOK_URL:
    secretKeyRef:
      name: alertmanager-slack-webhook
      key: webhook-url
```

`alertmanager-slack-webhook` Secret은 `infra/monitoring/secrets/resources/sealedsecret.yaml`의 SealedSecret으로
관리합니다. Grafana와 Alertmanager가 같은 `monitoring` namespace에 있으므로 같은 Secret을 재사용할 수 있습니다.
중복 알림을 줄이기 위해 예시 policy는 `alertname`, `service`, `env`, `severity`로 group하고
dev route의 `repeat_interval`은 `12h`, prod route placeholder의 `repeat_interval`은 `4h`로 둡니다.

contact point URL 예시:

```yaml
settings:
  url: $GRAFANA_ALERTING_SLACK_WEBHOOK_URL
```

Grafana file provisioning은 Grafana 시작 시 읽히며, 변경 사항은 Grafana 재시작 또는 Admin API hot reload가
필요합니다. ArgoCD로 Helm values 변경이 반영되면 Grafana pod가 새 provisioning 파일을 읽도록 rollout restart를
수행하는 방식이 가장 단순합니다.

적용 예시:

```bash
argocd app sync grafana
kubectl -n monitoring rollout restart deployment/grafana
```

Admin API로 provisioning을 hot reload할 수도 있지만, 운영에서는 ArgoCD sync 후 Grafana pod 재시작으로
파일 provisioning을 다시 읽히는 방식을 기본으로 사용합니다.
