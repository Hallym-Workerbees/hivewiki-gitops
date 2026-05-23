# Monitoring Alerts

Prometheus와 Alertmanager는 `prometheus-community/prometheus` Helm chart로 배포합니다.
`kube-prometheus-stack`은 사용하지 않습니다.

주요 설정 파일:

- ArgoCD Application: `argocd/infra/prometheus.yaml`
- Helm values: `infra/monitoring/prometheus/values.yaml`
- Slack webhook SealedSecret: `infra/monitoring/prometheus/sealedsecret.yaml`

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

## Excluded Alerts

`kube-green` alerts are intentionally excluded from Prometheus rules. kube-green does not currently expose the
needed metrics, so it should be handled separately with Loki log-based alerts.

`KarpenterUnschedulablePods` is not defined. The available Karpenter metrics do not include a direct unschedulable
pod count metric such as `karpenter_scheduler_unschedulable_pods_count`.

`RDSFreeStorageLow` is not defined. Current RDS metrics include free storage bytes, but not total or allocated storage,
so a safe free-space percentage cannot be calculated from Prometheus metrics alone.
