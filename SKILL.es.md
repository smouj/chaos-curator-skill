name: chaos-curator
version: 1.3.0
author: SMOUJBOT
description: Domina la entropía del sistema diseñando arquitecturas resilientes que prosperan en condiciones impredecibles
tags:
  - chaos
  - resilience
  - infrastructure
  - SRE
  - monitoring
dependencies:
  - kubectl >= 1.24
  - helm >= 3.8
  - aws-cli >= 2.8
  - azure-cli >= 2.50
  - jq >= 1.6
  - yq >= 4.30
  - prometheus >= 2.40
  - grafana-cli >= 9.0
  - chaos-mesh >= 2.5
  - litmusctl >= 0.14
required_env:
  - KUBECONFIG
  - AWS_PROFILE
  - AZURE_SUBSCRIPTION_ID
  - PROMETHEUS_URL
  - GRAFANA_TOKEN
  - SLACK_WEBHOOK_URL
  - CHAOS_EXPERIMENT_NAMESPACE
platforms:
  - linux
  - darwin
---

# Chaos Curator

## Propósito

Chaos Curator valida y mejora sistemáticamente la resiliencia del sistema mediante experimentos de caos controlados en infraestructura distribuida. Casos de uso reales:

- Validar circuit breakers de microservicios y lógica de reintentos cuando los objetivos upstream devuelven 500s con 2s de latencia
- Probar failover de alta disponibilidad de PostgreSQL cuando el nodo primario deja de responder durante 30s
- Verificar que Kubernetes HPA escala de 3 a 15 pods en 2 minutos bajo presión de CPU
- Confirmar que Redis Cluster sobrevive la pérdida del 50% de nodos manteniendo quórum
- Garantizar que el failover Multi-AZ de AWS RDS completa en 30 segundos con <1% de pérdida de transacciones
- Probar la resiliencia de mTLS de Istio cuando el CA intermedio expira
- Validar procedimientos de restauración de backups bajo carga similar a producción
- Medir RTO/RPO para flujos críticos durante outages a nivel de AZ

## Alcance

Chaos Curator proporciona estos comandos exactos:

```
chaos-curator experiment generate <template> --target <resource> --duration <time> --interval <check-frequency> --metrics-threshold <thresholds> --blast-radius <percentage>
chaos-curator experiment inject <experiment-id> --namespace <ns> --wait-for-completion --notify-slack --rollback-on-failure
chaos-curator experiment status <experiment-id> --watch --output json
chaos-curator experiment timeline <experiment-id> --format timeline
chaos-curator experiment abort <experiment-id> --grace-period <seconds> --force
chaos-curator validate infrastructure <cluster-name> --check-pod-disruption-budgets --check-hpa --check-pdb --check-network-policies
chaos-curator validate metrics <service-name> --query "rate(http_requests_total[5m])" --alert-rules --slo-baseline <value>
chaos-curator resilience score <namespace> --services <svc1,svc2> --include-chaos-metrics --report-format html
chaos-curator blast-radius calculate <experiment-yaml> --dependencies-from-istio --max-impact-percentage <value>
chaos-curator rollback create <experiment-id> --type helm --release-name <release> --namespace <ns> --timeout <seconds>
chaos-curator rollback execute <rollback-id> --verify-post-conditions --dry-run
chaos-curator dashboard open --experiment <id> --grafana-dashboard-id <id> --telemetry-channel <slack-channel>
chaos-curator hypothesis verify <hypothesis-id> --compare-baseline --statistical-significance <p-value>
```

Tipos de experimento soportados (acciones de caos reales):
- `pod-kill`: `kubectl delete pod --force --grace-period=0` con selector coincidente
- `network-delay`: `tc qdisc add` con `netem` causando `delay 100ms 20ms distribution normal`
- `network-loss`: `tc qdisc add` con `loss 10%`
- `network-corruption`: `tc qdisc add` con `corrupt 1%`
- `network-bandwidth`: `tc qdisc add` con `tbf rate 1mbit burst 32kbit latency 400ms`
- `io-stress`: `stress-ng --io 2 --io-method randwrite --timeout 30s`
- `cpu-stress`: `stress-ng --cpu 4 --cpu-method matrix --timeout 30s`
- `memory-stress`: `stress-ng --vm 2 --vm-bytes 1G --timeout 30s`
- `dns-chaos`: `dnsmasq` devolviendo NXDOMAIN o respuestas retardadas para dominios específicos
- `time-skew`: `date -s "+100 seconds"` dentro del contenedor con capability `SYS_TIME`
- `http-fault`: Inyección de fallos de Envoy/Istio con porcentajes `abort` o `delay`
- `gcp-zone-failure`: `gcloud compute instances delete` con `--zone` apuntando
- `aws-az-failure`: `aws ec2 describe-instances` + `terminate-instances` para AZ específica
- `azure-fault`: `az vm deallocate` para instancias VMSS en dominio de fallo específico
- `etcd-follower-down`: `systemctl stop etcd` en nodos seguidores con protección de líder
- `redis-master-failover`: `redis-cli -h <master-ip> debug segfault` con promoción de sentinel
- `postgres-replica-promotion`: `pg_ctl promote -D /var/lib/postgresql/data`
- `kafka-broker-down`: Detener proceso de broker kafka con verificación de gestión ISR
- `disk-fill`: `dd if=/dev/zero of=/tmp/fill bs=1M count=5000` hasta 95% de uso de disco

## Proceso de Trabajo Detallado

**Flujo de trabajo real para probar resiliencia de servicio de pagos (ejemplo E2E):**

```bash
# PASO 1: Verificaciones de seguridad pre-experimento
chaos-curator validate infrastructure payment-cluster \
  --check-pod-disruption-budgets \
  --check-hpa \
  --check-network-policies

# Salida esperada:
# ✓ PDB para payment-service: minAvailable 70% (3/10 pods)
# ✓ HPA payment-service: CPU objetivo 70%, actual 45%, minReplicas 5, maxReplicas 20
# ✓ NetworkPolicies restringen ingress a fuentes confiables únicamente
# ✓ Todos los servicios críticos tienen reglas de anti-afinidad
# ✓ Tamaño del pool de conexiones BD: 50 (con 30% de margen)
# ⚠️  Alerta para payment-service-error-rate > 1% configurada (actualmente 0.05%)
# ✓ Backup completado 2 horas atrás (restore test verificado semanalmente)

# PASO 2: Calcular radio de impacto antes de inyectar
chaos-curator blast-radius calculate <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-latency-test
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: payment-service
  delay:
    latency: "500ms"
    correlation: "100"
    jitter: "50ms"
  direction: to
  target:
    selector:
      namespaces:
        - production
      labelSelectors:
        app: card-processor
EOF

# Salida:
# Análisis de radio de impacto:
# - Afectados directamente: payment-service (1 deployment, 5 pods)
# - Dependencias downstream: card-processor (3 pods), fraud-detection (2 pods)
# - Impacto de tráfico estimado: 12% del volumen de pagos
# - Tolerancia máxima de error: 2% (brecha SLO a 3%)
# ✓ Radio de impacto dentro de límites aceptables (<15%)

# PASO 3: Generar experimento desde plantilla con restricciones
chaos-curator experiment generate network-delay \
  --target "app=payment-service,env=production" \
  --namespace production \
  --duration 10m \
  --latency 500ms \
  --jitter 50ms \
  --metrics-threshold '{"error_rate": "2%", "p99_latency": "2000ms", "circuit_breaker_open": "false"}' \
  --blast-radius 15% \
  --dry-run

# Salida (YAML):
# apiVersion: chaos-mesh.org/v1alpha1
# kind: NetworkChaos
# metadata:
#   name: payment-network-delay-20240309-abc123
#   annotations:
#     chaos-curator/slo-baseline: '{"error_rate":"0.5%","p99_latency":"800ms"}'
# spec:
#   action: delay
#   mode: one
#   selector:
#     namespaces: ["production"]
#     labelSelectors: {"app":"payment-service"}
#   delay:
#     latency: "500ms"
#     jitter: "50ms"
#   duration: "10m"
#   scheduler:
#     cron: "@every 10m"

# PASO 4: Inyectar con monitoreo y salvaguardas de rollback automático
chaos-curator experiment inject payment-network-delay-20240309-abc123 \
  --namespace production \
  --wait-for-completion \
  --notify-slack "#chaos-engine" \
  --rollback-on-failure \
  --pre-condition "chaos-curator validate metrics payment-service --query 'sum(rate(http_requests_total{status!~"5.."}[2m])) / sum(rate(http_requests_total[2m])) > 0.98'" \
  --post-condition "chaos-curator validate metrics payment-service --query 'sum(rate(circuit_breaker_state{state="OPEN"}[5m])) == 0'"

# Salida en tiempo real:
# Experimento iniciado: 2024-03-09T14:00:00Z
# Pre-condición superada: Tasa de error baseline 99.2% (umbral >98%)
# Dashboard Grafana monitoreando: http://grafana.internal/d/chaos/abc123
# Notificación Slack enviada: "Iniciando experimento de latencia de pagos (10m, 500ms delay)"
#
# T+03:42:
# ⚠️  Tasa de error se disparó a 2.1% (umbral 2%)
# ⚠️  P99 latencia: 1850ms (umbral 2000ms) - approaching limit
# ✓ Estado circuit breaker: CLOSED
#
# T+08:15:
# ✓ Todo dentro de umbrales
#
# Experimento completado: 2024-03-09T14:10:00Z
# ✓ Post-condición superada: Sin circuit breakers en estado OPEN
# ✓ Todos los servicios se recuperaron en 60s post-caos
# Reporte: /var/log/chaos-curator/payment-network-delay-20240309-abc123/report.json

# PASO 5: Generar reporte de puntuación de resiliencia
chaos-curator resilience score production \
  --services payment-service,card-processor,fraud-detection \
  --include-chaos-metrics \
  --report-format html \
  --output resilience-score-20240309.html

# Reporte incluye:
# - Mean Time To Recovery (MTTR): 47s
# - Consumo de error budget durante experimento: 0.8% del SLO mensual
# - Heatmap de impacto de dependencias de servicios
# - Conteo de activaciones de circuit breaker: 3 (payment-service)
# - Prevención de storm de reintentos: ✓ (tasa de reintentos se mantuvo <15%)
# - Puntuación general de resiliencia: 87/100 (+5 desde el mes pasado)

# PASO 6: Verificación de hipótesis con análisis estadístico
chaos-curator hypothesis verify paylat-h001 \
  --compare-baseline \
  --statistical-significance 0.05 \
  --metrics '{
    "observed": "rate(http_request_duration_seconds_bucket{le="1.0"}[10m])",
    "baseline": "rate(http_request_duration_seconds_bucket{le="1.0"}[10m]) offset 1h"
  }'

# Salida:
# Hipótesis H001: "El servicio de pagos mantiene <2% de tasa de error bajo latencia de 500ms"
# Test estadístico: Prueba U de Mann-Whitney
# p-valor: 0.032 (< umbral 0.05)
# Resultado: ✗ RECHAZADA - Tasa de error aumentó de 0.5% a 2.1% (estadísticamente significativo)
# Recomendación: Aumentar umbral de timeout de circuit breaker de 2s a 5s
```

## Reglas de Oro

1. **Nunca ejecutar caos en producción sin prueba de rollback documentada en staging el mismo día**. Ejecutar `chaos-curator rollback create --dry-run` y verificar que el rollback completa en <60s.
2. **Nunca exceder radio de impacto del 5% sin aprobación de director**. `chaos-curator blast-radius calculate` debe mostrar <5% de impacto; mayor requiere flag `--approved-by-cto`.
3. **Siempre verificar SLOs antes, durante, después**. Ejecutar `chaos-curator validate metrics` pre/post; experimento auto-aborta si brecha de SLO >2x umbral.
4. **Nunca ejecutar dos experimentos en la misma ruta de dependencia concurrentemente**. Chaos Curator mantiene archivo de bloqueo en `/var/lock/chaos-curator/<dependency-hash>.lock`.
5. **Siempre tener monitoreo humano**. `--notify-slack` requerido; tiempo de respuesta <5 minutos. Sin caos sin reconocimiento de SRE on-call en Slack.
6. **Ejecutar post-mortem por cada brecha de SLO**. Incluso si impacto <1%, generar reporte con `chaos-curator experiment timeline` y análisis 5 Whys.
7. **Nunca modificar infraestructura de producción fuera de experimentos de caos**. Todos los cambios deben ser via `chaos-curator experiment inject` con auditoría GitOps.
8. **Mantener runbook de ingeniería de caos**. Cada plantilla de experimento debe tener página de runbook en Confluence con prefijo "Chaos Playbook".
9. **Respetar períodos de blackout de negocio**. Chaos Curator lee var env `CHAOS_BLACKOUT_SCHEDULE`; experimentos bloqueados durante horas pico (08:00-20:00 local) a menos que flag `--emergency` con aprobación de VP.
10. **Verificar restaurabilidad de backups antes de caos de infraestructura**. Ejecutar `chaos-curator validate backup --test-restore` para BD involucradas en experimento.

## Ejemplos

**Ejemplo 1: Simular outage de AZ de AWS para deployment Multi-AZ**
```bash
# Calcular impacto primero
chaos-curator blast-radius calculate eks-node-termination.yaml \
  --dependencies-from-istio \
  --max-impact-percentage 10

# Salida:
# Objetivo: Grupo de nodos EKS prod-node-group-1 (us-east-1a)
# Pods a ser terminados: 23/120 (19%)
# Servicios impactados: api-gateway (5 pods), user-service (4 pods), session-store (3 pods)
# Tráfico afectado: 8% (basado en telemetría Istio)
# RTO máximo estimado: 90 segundos (escalado HPA)
# ✓ Dentro de límite de radio de impacto (10%)
# ✗ Falta: session-store no tiene replicación cross-AZ - PUNTO ÚNICO DE FALLO CRÍTICO

# Corregir punto único de fallo primero, luego proceder:
chaos-curator experiment inject eks-node-termination.yaml \
  --cluster prod-eks \
  --wait-for-completion \
  --notify-slack "#prod-chaos" \
  --pre-condition "chaos-curator validate metrics payment-service --query 'sum(rate(http_requests_total{status!~"5.."}[2m])) / sum(rate(http_requests_total[2m])) > 0.98'" \
  --rollback-on-failure

# Notificación Slack:
# [CHAOS] Iniciando eks-node-termination (cluster: prod-eks, AZ: us-east-1a)
# Monitoreo: http://grafana.company.com/d/chaos-az-failure/abc123
# Runbook: https://confluence.company.com/display/SRE/Chaos+Playbook+-+AZ+Failure
# Radio de impacto: 8% tráfico, 19% pods, 90s RTO
# [ACK] necesario de SRE on-call dentro de 5 minutos
```

**Ejemplo 2: Probar tolerancia a partición de Redis Cluster**
```bash
# Generar experimento de partición
chaos-curator experiment generate network-partition \
  --target "app.kubernetes.io/name=redis,role=master" \
  --partition-group "master-group" \
  --duration 5m \
  --metrics-threshold '{"quorum_loss": "false", "write_availability": "true", "data_loss": "0%"}' \
  > redis-partition.yaml

# Inyectar con rollback estricto
chaos-curator experiment inject redis-partition.yaml \
  --namespace caching \
  --wait-for-completion \
  --rollback-on-failure \
  --post-condition "kubectl exec -n caching deployment/redis-sentinel -- redis-cli cluster info | grep -q 'cluster_state:ok'"

# Salida de monitoreo en tiempo real:
# T+00:30: Partición de red establecida (maestro aislado de 2/5 réplicas)
# T+00:45: ✓ Sentinel electió nuevo maestro (tiempo de promoción: 12s)
# T+01:00: Disponibilidad de escritura: ✓ (quórum mantenido con 3/5 nodos)
# T+01:15: ⚠️  2 clientes desconectados (timeout de conexión)
# T+02:00: Partición curada automáticamente (red restaurada)
# T+02:30: Estado de cluster: OK (todos los nodos sincronizados)
# T+05:00: Experimento completado exitosamente
#
# Métricas de resiliencia:
# - Tiempo de failover: 12s (objetivo <30s)
# - Pérdida de datos: 0 bytes (quórum mantenido)
# - Disponibilidad de escritura: 100% durante partición
# - Impacto en clientes: 2% fallos de reconexión (umbral aceptable 5%)
```

**Ejemplo 3: Presión de CPU con validación HPA**
```bash
# Inyectar estrés de CPU apuntando a deployment específico
chaos-curator experiment inject <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: payment-cpu-stress
spec:
  mode: one
  selector:
    namespaces: ["production"]
    labelSelectors:
      app: payment-service
  stressers:
    cpu:
      workers: 4
      load: 80
      size: "200M"
  duration: "15m"
EOF \
  --namespace production \
  --wait-for-completion \
  --notify-slack "#chaos-eng" \
  --pre-condition "kubectl get hpa payment-service -o json | jq '.status.currentReplicas' -r" \
  --metrics-threshold '{"hpa_replicas": ">=5", "response_time_p95": "<2000ms"}'

# Escalado HPA observado:
# T+02:00: CPU avg: 78% (objetivo 70%)
# T+03:00: HPA escaló de 5 → 8 réplicas
# T+05:00: CPU avg: 72% (estable)
# T+12:00: Prueba de carga completada, CPU regresando a baseline
# T+13:00: HPA escaló abajo a 6 réplicas
# T+15:00: Experimento terminó, todos los pods saludables

# Verificación HPA:
# ✓ Tiempo de escalado-up: 3 minutos (objetivo <5 minutos)
# ✓ Terminación graceful en escalado-down: 2 minutos (respetando terminationGracePeriodSeconds)
# ✓ Sin disrupción de pods durante escalado (PDB respetado)
# ✓ Tiempo de respuesta se mantuvo <2000ms Throughout (máximo observado: 1450ms)
```

**Ejemplo 4: Aborto de emergencia y rollback**
```bash
# Ops detecta anomalía durante experimento
chaos-curator experiment status payment-network-delay-20240309-abc123

# Salida muestra brecha de umbral:
# ESTADO: EJECUTÁNDOSE
# MÉTRICAS ACTUALES:
# - error_rate: 3.2% (UMBRAL: 2%)
# - p99_latency: 3500ms (UMBRAL: 2000ms)
# - circuit_breaker_state: OPEN (3 instancias)
# ALERTA: Brecha de SLO detectada (3.2x umbral)

# Ejecutar abort de emergencia con rollback forzado
chaos-curator experiment abort payment-network-delay-20240309-abc123 \
  --grace-period 10 \
  --force \
  --notify-slack "#prod-alerts"

# Salida:
# Abortando experimento (ID: payment-network-delay-20240309-abc123)
# ✓ Recurso de caos eliminado del cluster
# ✓ Network policies restauradas desde backup
# ✓ Sidecars de service mesh reconfigurados
# ✓ Sincronización de tiempo corregida (drift NTP 0.5s → 0ms)
# ✓ Monitoreo: esperando normalización de métricas...
#
# Verificación post-abort:
# T+00:30: Tasa de error: 2.8% (aún elevada)
# T+01:00: Tasa de error: 1.2%
# T+02:00: Tasa de error: 0.6% (baseline restaurada)
# T+03:00: Todos los circuit breakers CLOSED
#
# ✓ Sistema regresó a baseline en 3 minutos
# ✓ No se requirió intervención manual
# ✓ Reporte de experimento incluirá línea de tiempo de abort
```

## Comandos de Rollback Específicos de Este Skill

**Rollback estándar** (generado automáticamente desde experimento):
```bash
chaos-curator rollback execute payment-network-delay-20240309-abc123 \
  --verify-post-conditions \
  --timeout 120s

# Realiza:
# 1. Restaurar recursos Kubernetes desde repo GitOps (flux/kustomize)
# 2. Revertir cambios de Istio VirtualService/DestinationRule
# 3. Resetear network policies a estado pre-experimento
# 4. Restaurar backups de ConfigMap/Secret de aplicación (tomados pre-experimento)
# 5. Terminar pods huérfanos de chaos-mesh
# 6. Validar que todos los servicios pasan health checks
# 7. Verificar que métricas regresaron a baseline (dentro de 5% de varianza)
```

**Escenarios de rollback manual** (cuando automático falla):

Escenario A: Releases Helm modificados durante experimento:
```bash
chaos-curator rollback create --type helm \
  --release-name payment-service \
  --namespace production \
  --snapshot-label chaos-exp-abc123 \
  --timeout 180s

# Crea plan de rollback:
# - helm rollback payment-service 3 (revisión antes de experimento)
# - Esperar completitud de rollout (timeout 180s)
# - Verificar pod readiness: kubectl wait --for=condition=ready pod -l app=payment-service
# - Ejecutar health check post-rollback: curl -f http://payment-service/health
```

Escenario B: Configuración de BD modificada:
```bash
# Restaurar postgresql.conf desde backup
POD=$(kubectl get pod -l app=postgres -n database -o jsonpath='{.items[0].metadata.name}')
kubectl cp /backups/chaos/abc123/postgresql.conf $POD:/var/lib/postgresql/data/postgresql.conf -n database
kubectl exec -n database $POD -- pg_ctl reload -D /var/lib/postgresql/data

# Verificar pool de conexiones restaurado
kubectl exec -n database $POD -- psql -c "SHOW max_connections;"
```

Escenario C: Recursos AWS terminados (no se pueden reiniciar):
```bash
# Reemplazar instancia terminada con nueva desde ASG
aws autoscaling set-instances-protection \
  --instance-ids i-0deadbeef12345678 \
  --no-protected-from-scale-in

aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-i-0deadbeef12345678 \
  --should-decrement-desired-capacity

# Esperar reemplazo (monitorear ASG desired vs actual)
watch -n 5 "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names prod-payment-asg | jq '.AutoScalingGroups[0].Instances[] | select(.LifecycleState=="InService") | .InstanceId'"
```

## Solución de Problemas Comunes

**Problema: Experimento atascado en estado "Pending"**
```bash
$ chaos-curator experiment status chaos-abc123
ESTADO: Pending (esperando scheduler)
Scheduler: "Cron expression */5 * * * *" - próxima ejecución en 3m 42s

# Razón: Scheduler configurado pero experimento no iniciado inmediatamente
# Solución: Esperar schedule o inyectar inmediatamente:
chaos-curator experiment inject chaos-abc123 --skip-scheduler

# O si scheduler es incorrecto:
chaos-curator experiment edit chaos-abc123 --patch '{"spec":{"schedule":"@now"}}'
```

**Problema: Falso positivo de umbral de métricas aborta experimento**
```bash
# Síntoma: Experimento abortado aunque servicios parecen saludables
# Razón: Prometheus query retornó NaN por inconsistencia de intervalo de scrape
chaos-curator experiment status chaos-abc123 | grep "abort-reason"
# Salida: "metrics-threshold: query returned no data (timeout 30s)"

# Solución:
# 1. Verificar que Prometheus está scrapeando target:
promtool query instant ${PROMETHEUS_URL} 'up{job="payment-service"}'
# 2. Incrementar timeout de query de umbrales:
chaos-curator experiment edit chaos-abc123 --patch '{"spec":{"metricsCheckTimeout":"60s"}}'
# 3. Ajustar query para considerar delay de scrape (usar offset):
#    rate(http_requests_total[5m] offset 1m) en vez de rate(http_requests_total[5m])
```

**Problema: Cálculo de radio de impacto incorrecto**
```bash
# Síntoma: Impacto real 30% pero cálculo mostró 5%
# Razón: Telemetría Istio no actualizada (dependencias cacheadas)

# Solución cálculo con telemetría en vivo:
chaos-curator blast-radius calculate experiment.yaml \
  --dependencies-from-istio \
  --use-live-telemetry \
  --telemetry-backend prometheus \
  --query 'istio_requests_total{destination_service=~"payment.*"}' \
  --lookback 10m

# Re-ejecutar con porcentaje de tráfico preciso de trazas actuales
```

**Problema: Rollback se queda en "Esperando completitud de rollout"**
```bash
# Síntoma: Rollback atascado en "Waiting for rollout to complete"
# Razón: Pod atascado en Terminating (foregroundDeleted timeout)

# Diagnosticar:
kubectl get pod -l app=payment-service -n production -o wide | grep Terminating

# Forzar delete de pods atascados (usar con precaución):
kubectl delete pod <pod-name> -n production --force --grace-period=0

# Si PVC previene delete:
kubectl patch deployment payment-service -n production --type='json' -p='[{"op":"remove","path":"/spec/template/spec/volumes/0"}]'

# Reanudar rollback:
chaos-curator rollback execute chaos-abc123 --continue
```

**Problema: Chaos Mesh controller no instala experimentos**
```bash
# Revisar pods de chaos-mesh:
kubectl get pods -n chaos-testing

# Causas comunes:
# 1. Permisos de cluster insuficientes:
kubectl auth can-i create chaosengines.chaos-mesh.org --namespace production
# Solución: Agregar ClusterRoleBinding para controller de chaos-mesh

# 2. MutatingAdmissionWebhook no configurado:
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations | grep chaos-mesh
# Solución: Instalar chaos-mesh con:
# helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh \
#   --namespace chaos-testing \
#   --set controllerManager.mutatingWebhook=true

# 3. Experimento esquema inválido:
chaos-curator experiment validate experiment.yaml
# Solucionar errores de schema, luego reintentar inyección
```

## Pasos de Verificación de Rollback

**Verificación automática post-rollback:**
```bash
chaos-curator rollback execute <exp-id> \
  --verify-post-conditions \
  --checks '
    - "kubectl get hpa payment-service -o jsonpath='{.status.currentReplicas}' == '5'"
    - "curl -s http://payment-service/health | jq -e '.status==\"UP\"'"
    - "kubectl get networkpolicy -n production -l chaos-experiment=<exp-id> | wc -l | grep '^0$'"
    - "kubectl get pods -l app=payment-service -o json | jq -r '.items[] | .status.phase' | grep -v Running | wc -l | grep '^0$'"
    - "promtool query instant ${PROMETHEUS_URL} 'sum(rate(http_requests_total{service="payment"}[2m])) > 1000'"
  '

# Todos los checks deben pasar con código de salida 0
# Si alguno falla, rollback marcado como PARCIAL y alerta PagerDuty disparada
```

**Comandos de verificación manual:**
```bash
# 1. Verificar que no quedan recursos de caos
kubectl get chaosengines,chaosexperiments,chaosresults -n production --field-selector metadata.name=chaos-abc123

# 2. Revisar network policies restauradas
kubectl get networkpolicy -n production -o yaml | grep -A5 -B5 "podSelector: {}"  # debería estar vacío (permitir todo por defecto en baseline)

# 3. Verificar HPA restaurado a baseline
kubectl get hpa payment-service -o json | jq '.spec.minReplicas, .spec.maxReplicas, .status.currentReplicas'

# 4. Confirmar que métricas baseline restauradas (dentro de 5% varianza):
baseline=$(promtool query instant ${PROMETHEUS_URL} 'avg(http_request_duration_seconds_sum{service="payment"}[1h] offset 1h)')
current=$(promtool query instant ${PROMETHEUS_URL} 'avg(http_request_duration_seconds_sum{service="payment"}[5m])')
variance=$(echo "scale=2; (($current - $baseline) / $baseline) * 100" | bc)
echo "Varianza: ${variance}%"
# Debería estar entre -5% y +5%

# 5. Verificar que no hay estrés residual:
kubectl top nodes | grep -v "^NAME" | awk '{print $5}' | grep -q "90%"
# No debería retornar nada (CPU <90% en todos los nodos)
```

**Checklist de rollback manual de emergencia:**
```bash
# Si rollback de chaos-curator falla:
[ ] kubectl delete chaosengine <id> --ignore-not-found
[ ] kubectl delete networkpolicy -l chaos-experiment=<id>
[ ] kubectl rollout undo deployment/payment-service
[ ] helm rollback payment-service <revision-before-chaos>
[ ] Restaurar configmaps: kubectl apply -f /backups/pre-chaos/configmaps.yaml
[ ] Restaurar secrets: kubectl apply -f /backups/pre-chaos/secrets.yaml (desencriptar primero)
[ ] Eliminar reglas iptables: (si se usa netem directamente)
[ ] Terminar pods de prueba huérfanos: kubectl delete pod -l purpose=chaos-test
[ ] Limpiar anotaciones de caos: kubectl annotate pods <pod> chaos-mesh/-
[ ] Verificar: chaos-curator validate post-check payment-service
[ ] Documentar incidente con línea de tiempo
```

## Variables de Entorno (Completo)

```bash
# Requeridas:
export KUBECONFIG=/etc/kubernetes/admin.conf
export AWS_PROFILE=prod-admin
export AZURE_SUBSCRIPTION_ID=xxxx-xxxx-xxxx-xxxx
export PROMETHEUS_URL=http://prometheus.monitoring.svc:9090
export GRAFANA_URL=https://grafana.company.com
export GRAFANA_TOKEN=eyJrZXl... (alcance read-only)
export SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx/yyy/zzz
export CHAOS_EXPERIMENT_NAMESPACE=chaos-testing
export CHAOS_MESH_VERSION=v2.5.0
export LITMUSCTL_NAMESPACE=litmus

# Opcionales con defaults:
export CHAOS_BLACKOUT_SCHEDULE="Mon-Fri 08:00-20:00 America/New_York"
export ROLLBACK_TIMEOUT=300
export EXPERIMENT_LOG_DIR=/var/log/chaos-curator
export METRICS_QUERY_TIMEOUT=30s
export SLO_BASELINE_LOOKBACK=1h
export BLOCK_DANGEROUS_EXPERIMENTS=true
export MAX_BLAST_RADIUS_PERCENT=5
export REQUIRED_PRE_CONDITIONS="db-backup-age<24h, pdb-minAvailable>0, hpa-configured=true"
export AUTO_ABORT_ON_SLO_BREACH=true
export SLO_BREACH_MULTIPLIER=2.0
export NOTIFY_ON_ABORT=true
export CHAOS_REPORT_RETENTION_DAYS=90
export GIT_OPS_REPO="/opt/gitops"
export RUNBOOK_URL_PREFIX="https://confluence.company.com/display/SRE/Chaos+Playboard+-"
```

## Dependencias (Instalación)

**Sistema:**
```bash
# Ubuntu/Debian
apt-get update && apt-get install -y \
  kubectl helm awscli jq yq prometheus \
  iptables iproute2 tc tcpdump stress-ng \
  curl wget gnupg

# Instalar chaos-mesh (Kubernetes operator)
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-testing \
  --create-namespace \
  --set controllerManager.logLevel=info \
  --set dashboard.securityMode=read-only

# Instalar Litmus (alternativa)
helm repo add litmus https://litmuschaos.github.io/litmus-helm/
helm install litmus litmus/litmus \
  --namespace litmus \
  --create-namespace
```

**Verificar:**
```bash
chaos-curator validate dependencies

# Salida esperada:
# ✓ kubectl: v1.24.0 (>=1.24)
# ✓ helm: v3.8.0 (>=3.8)
# ✓ aws-cli: v2.8.0 (>=2.8)
# ✓ jq: 1.6 (>=1.6)
# ✓ yq: 4.30.0 (>=4.30)
# ✓ promtool: 2.40.0 (>=2.40)
# ✓ chaos-mesh: pods ejecutándose en namespace chaos-testing
# ✓ Prometheus accesible en http://prometheus.monitoring.svc:9090
# ✓ Token de Grafana válido
# ✓ Slack webhook configurado
```

## Pasos de Verificación (Completo)

**1. Inspección de experimento dry-run:**
```bash
chaos-curator experiment generate network-delay \
  --target "app=order-service" \
  --duration 5m \
  --dry-run > /tmp/experiment.yaml

# Inspeccionar manualmente YAML generado para:
# - Selectores de objetivo correctos
# - Duración apropiada
# - Umbrales de métricas alineados con SLOs
# - Sin combinaciones peligrosas (ej. disk-fill en volumen root)

cat /tmp/experiment.yaml | chaos-curator experiment validate -
# Debería retornar: "VALID: Experiment YAML pasa validación de schema y chequeos de seguridad"
```

**2. Checklist pre-experimento:**
```bash
chaos-curator validate pre-experiment \
  --experiment-file /tmp/experiment.yaml \
  --cluster prod-eks \
  --namespace production

# Checklist (todos deben pasar):
# [✓] Antigüedad de backup < 24 horas
# [✓] Todos los servicios afectados tienen PDBs con minAvailable > 0
# [✓] HPA configurado y saludable
# [✓] Métricas de circuit breaker disponibles en Prometheus
# [✓] Reglas de alerta para servicios afectados existen y no están en estado firing
# [✓] Procedimiento de rollback probado en staging dentro de últimos 7 días
# [✓] SRE on-call reconoció (via reacción Slack)
# [✓] Radio de impacto < 5% (o excepción aprobada existe)
# [✓] Período de blackout de negocio no activo
# [✓] Duración de experimento < 30 minutos (a menos multi-hora aprobada)

echo $?
# Salida 0 = listo para proceder
# Salida 1 = checklist falló - no proceder
```

**3. Validación post-experimento:**
```bash
chaos-curator validate post-experiment \
  --experiment-id chaos-abc123 \
  --namespace production \
  --grace-period 300s \
  --metrics-window 10m

# Valida:
# - Todos los servicios retornaron a SLOs baseline (dentro de 5% varianza)
# - No hay recursos de caos residuales (chaosengines, networkpolicies)
# - No hay pods en CrashLoopBackOff o estado Error
# - Tasa de error retornó a < baseline * 1.1
# - Latencia P99 retornó a < baseline * 1.2
# - Todos los circuit breakers CLOSED
# - Conexiones de BD normalizadas
# - Auto-scaling estabilizado (conteo de réplicas dentro de 20% de baseline)

# Salida 0 = experimento exitoso, sistema saludable
# Salida 1 = degradación detectada - disparar respuesta de incidente
```

**4. Evaluación de madurez de ingeniería de caos:**
```bash
chaos-curator maturity score \
  --cluster prod-eks \
  --services payment,auth,user,notification \
  --include-infrastructure \
  --format json

# Retorna puntuación comprehensiva:
{
  "overall_score": 78,
  "categories": {
    "coverage": 85,
    "automation": 72,
    "safety": 90,
    "observability": 75,
    "recovery": 68
  },
  "recommendations": [
    "Incrementar cobertura de experimentos de caos para servicios stateful (actual: 40%)",
    "Automatizar pruebas de rollback (actualmente manual)",
    "Agregar auto-abort basado en SLO a 30% de experimentos",
    "Implementar automatización de runbook de caos para escenarios comunes"
  ]
}
```

**Smoke test:**
```bash
# Validación rápida de que Chaos Curator funciona end-to-end
kubectl create namespace chaos-smoke-test 2>/dev/null || true

cat > /tmp/smoke-exp.yaml <<'EOF'
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: smoke-cpu-stress
spec:
  mode: one
  selector:
    namespaces: ["chaos-smoke-test"]
    labelSelectors:
      app: nginx
  stressers:
    cpu:
      workers: 1
      load: 50
  duration: "30s"
EOF

chaos-curator experiment inject /tmp/smoke-exp.yaml \
  --namespace chaos-smoke-test \
  --wait-for-completion \
  --notify-slack ""  # deshabilitar Slack para smoke test

# Debería completarse exitosamente con código de salida 0
echo "Código de salida: $?"

# Limpieza
kubectl delete namespace chaos-smoke-test
```

## Notas Específicas de Plataforma

**Kubernetes:**
- Requiere chaos-mesh v2.5+ OR litmuschaos instalado
- Necesita ClusterRole con permisos para crear chaosengines, experiments, y delete pods
- Debe tener PodSecurityPolicy o PodSecurity Admission permitiendo contenedores privilegiados para netem/io-stress

**AWS:**
- Política IAM necesita `ec2:TerminateInstances`, `autoscaling:TerminateInstanceInAutoScalingGroup`
- SSM Session Manager opcional para ejecución remota de comandos
- Métricas CloudWatch requeridas para cálculo de radio de impacto

**Azure:**
- Rol Contributor en grupos de recursos objetivo
- Permisos para delete de instancias VMSS
- Métricas de Azure Monitor para radio de impacto

**GCP:**
- Rol Compute.instanceAdmin.v1
- Permisos de MIG (Managed Instance Group)
- API Cloud Monitoring metrics habilitada

**On-premises:**
- Acceso SSH a hosts físicos/virtuales
- tc/netem debe estar disponible (requiere capability NET_ADMIN)
- Stress-ng debe estar instalado en nodos objetivo (via DaemonSet)
```