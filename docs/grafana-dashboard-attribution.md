# Grafana 대시보드 템플릿 출처

이 문서는 이 저장소의 Grafana 대시보드 JSON 파일이 어떤 Grafana.com
대시보드 템플릿을 기반으로 작성되었는지 기록합니다.

저장소 루트의 `LICENSE` 파일은 HiveWiki GitOps 저장소를 위해 작성된
콘텐츠에 적용됩니다. 아래 대시보드 JSON 파일은 외부 Grafana 대시보드
템플릿을 기반으로 하므로, 원저작자가 Apache-2.0과 호환되는 라이선스를
명시적으로 제공하지 않는 한 이 저장소의 Apache-2.0 라이선스 허여 범위에
포함하지 않습니다.

이 출처 표시를 추가한 시점에 링크된 Grafana.com 대시보드 페이지에서는
명시적인 라이선스 고지를 확인하지 못했습니다. 따라서 아래 파일은 원저작자의
권리와 Grafana.com 이용 조건이 적용될 수 있는 제3자 템플릿 자료로
취급합니다.

## 대시보드 출처

| 로컬 파일 | 대시보드 | 출처 |
| --- | --- | --- |
| `infra/monitoring/grafana/dashboards/yace.json` | YACE | <https://grafana.com/grafana/dashboards/21327-yace2/> |
| `infra/monitoring/grafana/dashboards/k8s-logs.json` | Loki Kubernetes Logs | <https://grafana.com/grafana/dashboards/15141-kubernetes> |
| `infra/monitoring/grafana/dashboards/argocd.json` | ArgoCD | <https://grafana.com/grafana/dashboards/14584-argocd/> |
| `infra/monitoring/grafana/dashboards/k8s-dashboard.json` | K8S Dashboard | <https://grafana.com/grafana/dashboards/15661-k8s-dashbo> |
| `infra/monitoring/grafana/dashboards/karpenter.json` | Karpenter | <https://grafana.com/grafana/dashboards/20398-karpenter/> |

## 라이선스 처리 기준

- 위 대시보드 JSON 파일을 수정하거나 교체할 때 원본 출처 URL을 유지합니다.
- 위 제3자 대시보드 템플릿을 HiveWiki의 독자 저작물로 표시하지 않습니다.
- 추후 원저작자의 호환 라이선스가 확인되어 이 문서에 기록되기 전까지,
  위 대시보드 템플릿의 재사용 근거로 이 저장소의 Apache-2.0 라이선스를
  사용하지 않습니다.
