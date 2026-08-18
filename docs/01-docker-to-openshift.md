# Docker から OpenShift

| Docker | OpenShift |
|---|---|
| docker run | Deployment / Pod |
| -p | Service |
| named volume | PVC + volumeMount |
| docker exec | oc exec |
| docker logs | oc logs |
| reverse proxy | Route |

Deployment は期待状態を維持し、Pod は一時的な実行単位です。