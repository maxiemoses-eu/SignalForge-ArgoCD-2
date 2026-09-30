# SignalForge 2026: Network-Policies Platform Defaults
# Override per environment in helm-values/{env}/platform/network-policies.yaml.

# Port alignment: 5003 for Go Services, 8080 for API Gateway
# 2026 Logic: Using a list allows the firewall to open for ALL app types.
appPorts: 
  - 5003
  - 8080

# Primary port used for generic scrapes if not specified
appPort: 5003 

# Namespace isolation
ingressNamespace: ingress-nginx
monitoringNamespace: monitoring

# Infra pod selector labels (Bitnami Standard)
infra:
  postgres:
    labelKey: "app.kubernetes.io/name"
    labelValue: postgresql
    port: 5432
  redis:
    labelKey: "app.kubernetes.io/name"
    labelValue: redis
    port: 6379
  rabbitmq:
    labelKey: "app.kubernetes.io/name"
    labelValue: rabbitmq
    port: 5672