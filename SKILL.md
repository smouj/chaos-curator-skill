---
name: chaos-curator
version: 1.3.0
author: SMOUJBOT
description: Tames system entropy by designing resilient architectures that thrive under unpredictable conditions
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

## Purpose

Chaos Curator systematically validates and improves system resilience through controlled chaos experiments in distributed infrastructure. Real use cases:

- Validate microservice circuit breakers and retry logic when upstream targets return 500s with 2s latency
- Test PostgreSQL HA failover when primary node becomes unresponsive for 30s
- Verify Kubernetes HPA scales from 3 to 15 pods within 2 minutes under CPU pressure
- Confirm Redis Cluster survives 50% node loss while maintaining quorum
- Ensure AWS RDS Multi-AZ failover completes within 30 seconds with <1% transaction loss
- Test Istio mTLS resilience when intermediate CA expires
- Validate backup restoration procedures under production-like load
- Measure RTO/RPO for critical workflows during AZ-level outages

## Scope

Chaos Curator provides these exact commands:

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

Supported experiment types (real chaos actions):
- `pod-kill`: `kubectl delete pod --force --grace-period=0` with selector matching
- `network-delay`: `tc qdisc add` with `netem` causing `delay 100ms 20ms distribution normal`
- `network-loss`: `tc qdisc add` with `loss 10%`
- `network-corruption`: `tc qdisc add` with `corrupt 1%`
- `network-bandwidth`: `tc qdisc add` with `tbf rate 1mbit burst 32kbit latency 400ms`
- `io-stress`: `stress-ng --io 2 --io-method randwrite --timeout 30s`
- `cpu-stress`: `stress-ng --cpu 4 --cpu-method matrix --timeout 30s`
- `memory-stress`: `stress-ng --vm 2 --vm-bytes 1G --timeout 30s`
- `dns-chaos`: `dnsmasq` returning NXDOMAIN or delayed responses for specific domains
- `time-skew`: `date -s "+100 seconds"` inside container with `SYS_TIME` capability
- `http-fault`: Envoy/Istio fault injection with `abort` or `delay` percentages
- `gcp-zone-failure`: `gcloud compute instances delete` with `--zone` targeting
- `aws-az-failure`: `aws ec2 describe-instances` + `terminate-instances` for specific AZ
- `azure-fault`: `az vm deallocate` for VMSS instances in specific fault domain
- `etcd-follower-down`: `systemctl stop etcd` on follower nodes with leader protection
- `redis-master-failover`: `redis-cli -h <master-ip> debug segfault` with sentinel promotion
- `postgres-replica-promotion`: `pg_ctl promote -D /var/lib/postgresql/data`
- `kafka-broker-down`: Stop kafka broker process with ISR management verification
- `disk-fill`: `dd if=/dev/zero of=/tmp/fill bs=1M count=5000` until 95% disk usage

## Detailed Work Process

**Real workflow for testing payment service resilience (E2E example):**

```bash
# STEP 1: Pre-experiment safety checks
chaos-curator validate infrastructure payment-cluster \
  --check-pod-disruption-budgets \
  --check-hpa \
  --check-network-policies

# Expected output:
# ✓ PDB for payment-service: minAvailable 70% (3/10 pods)
# ✓ HPA payment-service: target CPU 70%, current 45%, minReplicas 5, maxReplicas 20
# ✓ NetworkPolicies restrict ingress to trusted sources only
# ✓ All critical services have anti-affinity rules
# ✓ Database connection pool size: 50 (with 30% headroom)
# ⚠️  Alert for payment-service-error-rate > 1% configured (currently 0.05%)
# ✓ Backup completed 2 hours ago (verified restore test weekly)

# STEP 2: Calculate blast radius before injecting
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

# Output:
# Blast radius analysis:
# - Directly affected: payment-service (1 deployment, 5 pods)
# - Downstream dependencies: card-processor (3 pods), fraud-detection (2 pods)
# - Estimated traffic impact: 12% of payment volume
# - Max error rate tolerance: 2% (SLO breach at 3%)
# ✓ Blast radius within acceptable limits (<15%)

# STEP 3: Generate experiment from template with constraints
chaos-curator experiment generate network-delay \
  --target "app=payment-service,env=production" \
  --namespace production \
  --duration 10m \
  --latency 500ms \
  --jitter 50ms \
  --metrics-threshold '{"error_rate": "2%", "p99_latency": "2000ms", "circuit_breaker_open": "false"}' \
  --blast-radius 15% \
  --dry-run

# Output (YAML):
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

# STEP 4: Inject with monitoring and automatic rollback safeguards
chaos-curator experiment inject payment-network-delay-20240309-abc123 \
  --namespace production \
  --wait-for-completion \
  --notify-slack "#chaos-engine" \
  --rollback-on-failure \
  --pre-condition "chaos-curator validate metrics payment-service --query 'sum(rate(http_requests_total{status!~\"5..\"}[2m])) / sum(rate(http_requests_total[2m])) > 0.98'" \
  --post-condition "chaos-curator validate metrics payment-service --query 'sum(rate(circuit_breaker_state{state=\"OPEN\"}[5m])) == 0'"

# Real-time output:
# Experiment started: 2024-03-09T14:00:00Z
# Pre-condition passed: Error rate baseline 99.2% (threshold >98%)
# Monitoring Grafana dashboard: http://grafana.internal/d/chaos/abc123
# Slack notification sent: "Starting payment latency experiment (10m, 500ms delay)"
# 
# T+03:42:
# ⚠️  Error rate spiked to 2.1% (threshold 2%)
# ⚠️  P99 latency: 1850ms (threshold 2000ms) - approaching limit
# ✓ Circuit breaker state: CLOSED
# 
# T+08:15:
# ✓ Everything within thresholds
# 
# Experiment completed: 2024-03-09T14:10:00Z
# ✓ Post-condition passed: No circuit breakers in OPEN state
# ✓ All services recovered within 60s post-chaos
# Report: /var/log/chaos-curator/payment-network-delay-20240309-abc123/report.json

# STEP 5: Generate resilience score report
chaos-curator resilience score production \
  --services payment-service,card-processor,fraud-detection \
  --include-chaos-metrics \
  --report-format html \
  --output resilience-score-20240309.html

# Report includes:
# - Mean Time To Recovery (MTTR): 47s
# - Error budget burn during experiment: 0.8% of monthly SLO
# - Service dependency impact heatmap
# - Circuit breaker activation count: 3 (payment-service)
# - Retry storm prevention: ✓ (retry rate stayed <15%)
# - Overall resilience score: 87/100 (+5 from last month)

# STEP 6: Hypothesis verification with statistical analysis
chaos-curator hypothesis verify paylat-h001 \
  --compare-baseline \
  --statistical-significance 0.05 \
  --metrics '{
    "observed": "rate(http_request_duration_seconds_bucket{le=\"1.0\"}[10m])",
    "baseline": "rate(http_request_duration_seconds_bucket{le=\"1.0\"}[10m]) offset 1h"
  }'

# Output:
# Hypothesis H001: "Payment service maintains <2% error rate under 500ms latency"
# Statistical test: Mann-Whitney U test
# p-value: 0.032 (< 0.05 threshold)
# Result: ✗ REJECTED - Error rate increased from 0.5% to 2.1% (statistically significant)
# Recommendation: Increase circuit breaker timeout threshold from 2s to 5s
```

## Golden Rules

1. **Never run chaos in production without a documented rollback test in staging same day**. Execute `chaos-curator rollback create --dry-run` and verify rollback completes <60s.
2. **Never exceed blast radius of 5% without director approval**. `chaos-curator blast-radius calculate` must show <5% impact; larger requires `--approved-by-cto` flag.
3. **Always verify SLOs before, during, after**. Run `chaos-curator validate metrics` pre/post; experiment auto-aborts if SLO breach >2x threshold.
4. **Never run two experiments on same dependency path concurrently**. Chaos Curator maintains experiment lock file at `/var/lock/chaos-curator/<dependency-hash>.lock`.
5. **Always have human monitoring**. `--notify-slack` required; response time <5 minutes. No chaos without on-call SRE acknowledgment in Slack.
6. **Run post-mortem for every SLO breach**. Even if <1% impact, generate report with `chaos-curator experiment timeline` and 5 Whys analysis.
7. **Never modify production infrastructure outside of chaos experiments**. All changes must be via `chaos-curator experiment inject` with GitOps audit trail.
8. **Maintain chaos engineering runbook**. Each experiment template must have corresponding runbook page in Confluence with "Chaos Playbook" prefix.
9. **Respect business blackout periods**. Chaos Curator reads `CHAOS_BLACKOUT_SCHEDULE` env var; experiments blocked during peak hours (08:00-20:00 local) unless `--emergency` flag with VP approval.
10. **Verify backup restorability before infrastructure chaos**. Run `chaos-curator validate backup --test-restore` for databases involved in experiment.

## Examples

**Example 1: Simulate AWS AZ outage for multi-AZ deployment**
```bash
# Calculate impact first
chaos-curator blast-radius calculate eks-node-termination.yaml \
  --dependencies-from-istio \
  --max-impact-percentage 10

# Output:
# Target: EKS node group prod-node-group-1 (us-east-1a)
# Pods to be terminated: 23/120 (19%)
# Services impacted: api-gateway (5 pods), user-service (4 pods), session-store (3 pods)
# Traffic affected: 8% (based on Istio telemetry)
# Max RTO estimated: 90 seconds (HPA scale-up)
# ✓ Within blast radius limit (10%)
# ✗ Missing: session-store has no cross-AZ replication - CRITICAL SINGLE POINT OF FAILURE

# Fix single point of failure first, then proceed:
chaos-curator experiment inject eks-node-termination.yaml \
  --cluster prod-eks \
  --wait-for-completion \
  --notify-slack "#prod-chaos" \
  --pre-condition "kubectl get pdb -n production -o json | jq '.items[] | select(.metadata.name==\"session-store-pdb\") | .status.disruptionsAllowed' == '0'" \
  --rollback-on-failure

# Slack notification:
# [CHAOS] Starting eks-node-termination (cluster: prod-eks, AZ: us-east-1a)
# Monitor: http://grafana.company.com/d/chaos-az-failure/abc123
# Runbook: https://confluence.company.com/display/SRE/Chaos+Playbook+-+AZ+Failure
# Blast radius: 8% traffic, 19% pods, 90s RTO
# [ACK] needed from on-call SRE within 5 minutes
```

**Example 2: Test Redis Cluster partition tolerance**
```bash
# Generate partition experiment
chaos-curator experiment generate network-partition \
  --target "app.kubernetes.io/name=redis,role=master" \
  --partition-group "master-group" \
  --duration 5m \
  --metrics-threshold '{"quorum_loss": "false", "write_availability": "true", "data_loss": "0%"}' \
  > redis-partition.yaml

# Inject with strict rollback
chaos-curator experiment inject redis-partition.yaml \
  --namespace caching \
  --wait-for-completion \
  --rollback-on-failure \
  --post-condition "kubectl exec -n caching deployment/redis-sentinel -- redis-cli cluster info | grep -q 'cluster_state:ok'"

# Real-time monitoring output:
# T+00:30: Network partition established (master isolated from 2/5 replicas)
# T+00:45: ✓ Sentinel elected new master (promotion time: 12s)
# T+01:00: Write availability: ✓ (quorum maintained with 3/5 nodes)
# T+01:15: ⚠️  2 clients disconnected (connection timeout)
# T+02:00: Partition healed automatically (network restored)
# T+02:30: Cluster state: OK (all nodes synchronized)
# T+05:00: Experiment completed successfully
# 
# Resilience metrics:
# - Failover time: 12s (target <30s)
# - Data loss: 0 bytes (quorum maintained)
# - Write availability: 100% during partition
# - Client impact: 2% reconnection failures (acceptable threshold 5%)
```

**Example 3: CPU pressure with HPA validation**
```bash
# Inject CPU stress targeting specific deployment
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

# HPA scaling observed:
# T+02:00: CPU avg: 78% (target 70%)
# T+03:00: HPA scaled from 5 → 8 replicas
# T+05:00: CPU avg: 72% (stable)
# T+12:00: Load test completed, CPU returning to baseline
# T+13:00: HPA scaled down to 6 replicas
# T+15:00: Experiment ended, all pods healthy

# HPA verification:
# ✓ Scale-up time: 3 minutes (target <5 minutes)
# ✓ Scale-down graceful termination: 2 minutes (respect terminationGracePeriodSeconds)
# ✓ No pod disruption during scaling (PDB respected)
# ✓ Response time stayed <2000ms throughout (max observed: 1450ms)
```

**Example 4: Emergency abort and rollback**
```bash
# Ops detects anomaly during experiment
chaos-curator experiment status payment-network-delay-20240309-abc123

# Output shows threshold breach:
# STATUS: RUNNING
# CURRENT METRICS:
# - error_rate: 3.2% (THRESHOLD: 2%)
# - p99_latency: 3500ms (THRESHOLD: 2000ms)
# - circuit_breaker_state: OPEN (3 instances)
# ALERT: SLO breach detected (3.2x threshold)

# Execute emergency abort with forced rollback
chaos-curator experiment abort payment-network-delay-20240309-abc123 \
  --grace-period 10 \
  --force \
  --notify-slack "#prod-alerts"

# Output:
# Aborting experiment (ID: payment-network-delay-20240309-abc123)
# ✓ Chaos resource deleted from cluster
# ✓ Network policies restored from backup
# ✓ Service mesh sidecars reconfigured
# ✓ Time synchronization corrected (NTP drift 0.5s → 0ms)
# ✓ Monitoring: waiting for metrics normalization...
# 
# Verification post-abort:
# T+00:30: Error rate: 2.8% (still elevated)
# T+01:00: Error rate: 1.2%
# T+02:00: Error rate: 0.6% (baseline restored)
# T+03:00: Circuit breakers all CLOSED
# 
# ✓ System returned to baseline within 3 minutes
# ✓ No manual intervention required
# ✓ Experiment report will include abort timeline
```

## Rollback Commands Specific to This Skill

**Standard rollback** (automatically generated from experiment):
```bash
chaos-curator rollback execute payment-network-delay-20240309-abc123 \
  --verify-post-conditions \
  --timeout 120s

# Performs:
# 1. Restore Kubernetes resources from GitOps repo (flux/kustomize)
# 2. Revert Istio VirtualService/DestinationRule changes
# 3. Reset network policies to pre-experiment state
# 4. Restore application ConfigMap/Secret backups (taken pre-experiment)
# 5. Terminate any orphaned pods from chaos-mesh
# 6. Validate all services passing health checks
# 7. Verify metrics returned to baseline (within 5% variance)
```

**Manual rollback scenarios** (when automatic fails):

Scenario A: Helm release modified during experiment:
```bash
chaos-curator rollback create --type helm \
  --release-name payment-service \
  --namespace production \
  --snapshot-label chaos-exp-abc123 \
  --timeout 180s

# Creates rollback plan:
# - helm rollback payment-service 3 (revision before experiment)
# - Wait for rollout completion (timeout 180s)
# - Verify pod readiness: kubectl wait --for=condition=ready pod -l app=payment-service
# - Run post-rollback health check: curl -f http://payment-service/health
```

Scenario B: Database configuration changed:
```bash
# Restore postgresql.conf from backup
POD=$(kubectl get pod -l app=postgres -n database -o jsonpath='{.items[0].metadata.name}')
kubectl cp /backups/chaos/abc123/postgresql.conf $POD:/var/lib/postgresql/data/postgresql.conf -n database
kubectl exec -n database $POD -- pg_ctl reload -D /var/lib/postgresql/data

# Verify connection pool restored
kubectl exec -n database $POD -- psql -c "SHOW max_connections;"
```

Scenario C: AWS resources terminated (can't restart):
```bash
# Replace terminated instance with fresh one from ASG
aws autoscaling set-instances-protection \
  --instance-ids i-0deadbeef12345678 \
  --no-protected-from-scale-in

aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-i-0deadbeef12345678 \
  --should-decrement-desired-capacity

# Wait for replacement (monitor ASG desired vs actual)
watch -n 5 "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names prod-payment-asg | jq '.AutoScalingGroups[0].Instances[] | select(.LifecycleState==\"InService\") | .InstanceId'"
```

## Troubleshooting Common Issues

**Issue: Experiment stuck in "Pending" state**
```bash
$ chaos-curator experiment status chaos-abc123
STATUS: Pending (waiting for scheduler)
Scheduler: "Cron expression */5 * * * *" - next run in 3m 42s

# Reason: Scheduler configured but experiment not started immediately
# Fix: Either wait for schedule or inject immediately:
chaos-curator experiment inject chaos-abc123 --skip-scheduler

# OR if scheduler is wrong:
chaos-curator experiment edit chaos-abc123 --patch '{"spec":{"schedule":"@now"}}'
```

**Issue: Metrics threshold false positive abort**
```bash
# Symptom: Experiment aborted even though services appear healthy
# Reason: Prometheus query returned NaN due to scrape interval mismatch
chaos-curator experiment status chaos-abc123 | grep "abort-reason"
# Output: "metrics-threshold: query returned no data (timeout 30s)"

# Fix:
# 1. Verify Prometheus is scraping target:
promtool query instant ${PROMETHEUS_URL} 'up{job="payment-service"}'
# 2. Increase thresholds query timeout:
chaos-curator experiment edit chaos-abc123 --patch '{"spec":{"metricsCheckTimeout":"60s"}}'
# 3. Adjust query to account for scrape delay (use offset):
#    rate(http_requests_total[5m] offset 1m) instead of rate(http_requests_total[5m])
```

**Issue: Blast radius calculation incorrect**
```bash
# Symptom: Actual impact 30% but calculation showed 5%
# Reason: Istio telemetry not updated (cached dependencies)

# Fix calculation with live telemetry:
chaos-curator blast-radius calculate experiment.yaml \
  --dependencies-from-istio \
  --use-live-telemetry \
  --telemetry-backend prometheus \
  --query 'istio_requests_total{destination_service=~"payment.*"}' \
  --lookback 10m

# Re-run with accurate traffic percentage from actual traces
```

**Issue: Rollback hangs on "Waiting for rollout completion"**
```bash
# Symptom: Rollback stuck at "Waiting for rollout to complete"
# Reason: Pod stuck in Terminating (foregroundDeleted timeout)

# Diagnose:
kubectl get pod -l app=payment-service -n production -o wide | grep Terminating

# Force delete stuck pods (use with caution):
kubectl delete pod <pod-name> -n production --force --grace-period=0

# If PVC prevents deletion:
kubectl patch deployment payment-service -n production --type='json' -p='[{"op":"remove","path":"/spec/template/spec/volumes/0"}]'

# Resume rollback:
chaos-curator rollback execute chaos-abc123 --continue
```

**Issue: Chaos Mesh controller not installing experiments**
```bash
# Check chaos-mesh pods:
kubectl get pods -n chaos-testing

# Common causes:
# 1. Insufficient cluster permissions:
kubectl auth can-i create chaosengines.chaos-mesh.org --namespace production
# Fix: Add ClusterRoleBinding for chaos-mesh controller

# 2. MutatingAdmissionWebhook not configured:
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations | grep chaos-mesh
# Fix: Install chaos-mesh with:
# helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh \
#   --namespace chaos-testing \
#   --set controllerManager.mutatingWebhook=true

# 3. Experiment invalid schema:
chaos-curator experiment validate experiment.yaml
# Fix schema errors, then retry injection
```

## Rollback Verification Steps

**Automated verification after rollback:**
```bash
chaos-curator rollback execute <exp-id> \
  --verify-post-conditions \
  --checks '
    - "kubectl get hpa payment-service -o jsonpath='{.status.currentReplicas}' == '5'"
    - "curl -s http://payment-service/health | jq -e '.status==\"UP\"'"
    - "kubectl get networkpolicy -n production -l chaos-experiment=<exp-id> | wc -l | grep '^0$'"
    - "kubectl get pods -l app=payment-service -o json | jq -r '.items[] | .status.phase' | grep -v Running | wc -l | grep '^0$'"
    - "promtool query instant ${PROMETHEUS_URL} 'sum(rate(http_requests_total{service=\"payment\"}[2m])) > 1000'"
  '

# All checks must pass with exit code 0
# If any fail, rollback marked as PARTIAL and PagerDuty alert triggered
```

**Manual verification commands:**
```bash
# 1. Verify no chaos resources remain
kubectl get chaosengines,chaosexperiments,chaosresults -n production --field-selector metadata.name=chaos-abc123

# 2. Check network policies restored
kubectl get networkpolicy -n production -o yaml | grep -A5 -B5 "podSelector: {}"  # should be empty (allow all by default in baseline)

# 3. Verify HPA restored to baseline
kubectl get hpa payment-service -o json | jq '.spec.minReplicas, .spec.maxReplicas, .status.currentReplicas'

# 4. Confirm metrics baseline restored (within 5% variance):
baseline=$(promtool query instant ${PROMETHEUS_URL} 'avg(http_request_duration_seconds_sum{service="payment"}[1h] offset 1h)')
current=$(promtool query instant ${PROMETHEUS_URL} 'avg(http_request_duration_seconds_sum{service="payment"}[5m])')
variance=$(echo "scale=2; (($current - $baseline) / $baseline) * 100" | bc)
echo "Variance: ${variance}%"
# Should be between -5% and +5%

# 5. Verify no residual stress:
kubectl top nodes | grep -v "^NAME" | awk '{print $5}' | grep -q "90%"
# Should return nothing (CPU <90% on all nodes)
```

**Emergency manual rollback checklist:**
```bash
# If chaos-curator rollback fails:
[ ] kubectl delete chaosengine <id> --ignore-not-found
[ ] kubectl delete networkpolicy -l chaos-experiment=<id>
[ ] kubectl rollout undo deployment/payment-service
[ ] helm rollback payment-service <revision-before-chaos>
[ ] Restore configmaps: kubectl apply -f /backups/pre-chaos/configmaps.yaml
[ ] Restore secrets: kubectl apply -f /backups/pre-chaos/secrets.yaml (decrypt first)
[ ] Remove iptables rules: (if using netem directly)
[ ] Terminate orphaned test pods: kubectl delete pod -l purpose=chaos-test
[ ] Clear chaos annotations: kubectl annotate pods <pod> chaos-mesh/-
[ ] Verify: chaos-curator validate post-check payment-service
[ ] Document incident with timeline
```

## Environment Variables (Complete)

```bash
# Required:
export KUBECONFIG=/etc/kubernetes/admin.conf
export AWS_PROFILE=prod-admin
export AZURE_SUBSCRIPTION_ID=xxxx-xxxx-xxxx-xxxx
export PROMETHEUS_URL=http://prometheus.monitoring.svc:9090
export GRAFANA_URL=https://grafana.company.com
export GRAFANA_TOKEN=eyJrZXl... (read-only scope)
export SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx/yyy/zzz
export CHAOS_EXPERIMENT_NAMESPACE=chaos-testing
export CHAOS_MESH_VERSION=v2.5.0
export LITMUSCTL_NAMESPACE=litmus

# Optional with defaults:
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

## Dependencies (Installation)

**System:**
```bash
# Ubuntu/Debian
apt-get update && apt-get install -y \
  kubectl helm awscli jq yq prometheus \
  iptables iproute2 tc tcpdump stress-ng \
  curl wget gnupg

# Install chaos-mesh (Kubernetes operator)
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-testing \
  --create-namespace \
  --set controllerManager.logLevel=info \
  --set dashboard.securityMode=read-only

# Install Litmus (alternative)
helm repo add litmus https://litmuschaos.github.io/litmus-helm/
helm install litmus litmus/litmus \
  --namespace litmus \
  --create-namespace
```

**Verify:**
```bash
chaos-curator validate dependencies

# Expected output:
# ✓ kubectl: v1.24.0 (>=1.24)
# ✓ helm: v3.8.0 (>=3.8)
# ✓ aws-cli: v2.8.0 (>=2.8)
# ✓ jq: 1.6 (>=1.6)
# ✓ yq: 4.30.0 (>=4.30)
# ✓ promtool: 2.40.0 (>=2.40)
# ✓ chaos-mesh: pods running in namespace chaos-testing
# ✓ Prometheus reachable at http://prometheus.monitoring.svc:9090
# ✓ Grafana token valid
# ✓ Slack webhook configured
```

## Verification Steps (Complete)

**1. Dry-run experiment inspection:**
```bash
chaos-curator experiment generate network-delay \
  --target "app=order-service" \
  --duration 5m \
  --dry-run > /tmp/experiment.yaml

# Manually inspect generated YAML for:
# - Correct target selectors
# - Appropriate duration
# - Metrics thresholds aligned with SLOs
# - No dangerous combinations (e.g., disk-fill on root volume)

cat /tmp/experiment.yaml | chaos-curator experiment validate -
# Should output: "VALID: Experiment YAML passes schema validation and safety checks"
```

**2. Pre-experiment checklist:**
```bash
chaos-curator validate pre-experiment \
  --experiment-file /tmp/experiment.yaml \
  --cluster prod-eks \
  --namespace production

# Checklist (all must pass):
# [✓] Backup age < 24 hours
# [✓] All affected services have PDBs with minAvailable > 0
# [✓] HPA configured and healthy
# [✓] Circuit breaker metrics available in Prometheus
# [✓] Alert rules for affected services exist and not in firing state
# [✓] Rollback procedure tested in staging within last 7 days
# [✓] On-call SRE acknowledged (via Slack reaction)
# [✓] Blast radius < 5% (or approved exception exists)
# [✓] Business blackout period not active
# [✓] Experiment duration < 30 minutes (unless multi-hour approved)

echo $?
# Exit 0 = ready to proceed
# Exit 1 = checklist failed - do not proceed
```

**3. Post-experiment validation:**
```bash
chaos-curator validate post-experiment \
  --experiment-id chaos-abc123 \
  --namespace production \
  --grace-period 300s \
  --metrics-window 10m

# Validates:
# - All services returned to baseline SLOs (within 5% variance)
# - No residual chaos resources (chaosengines, networkpolicies)
# - No pods in CrashLoopBackOff or Error state
# - Error rate returned to < baseline * 1.1
# - Latency P99 returned to < baseline * 1.2
# - Circuit breakers all CLOSED
# - Database connections normalized
# - Auto-scaling stabilized (replica count within 20% of baseline)

# Exit 0 = experiment successful, system healthy
# Exit 1 = degradation detected - trigger incident response
```

**4. Chaos engineering maturity assessment:**
```bash
chaos-curator maturity score \
  --cluster prod-eks \
  --services payment,auth,user,notification \
  --include-infrastructure \
  --format json

# Returns comprehensive score:
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
    "Increase chaos experiment coverage for stateful services (current: 40%)",
    "Automate rollback testing (currently manual)",
    "Add SLO-based auto-abort to 30% of experiments",
    "Implement chaos runbook automation for common scenarios"
  ]
}
```

**Smoke test:**
```bash
# Quick validation that Chaos Curator works end-to-end
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
  --notify-slack ""  # disable Slack for smoke test

# Should complete successfully with exit code 0
echo "Exit code: $?"

# Cleanup
kubectl delete namespace chaos-smoke-test
```

## Platform-Specific Notes

**Kubernetes:**
- Requires chaos-mesh v2.5+ OR litmuschaos installed
- Needs ClusterRole with permissions to create chaosengines, experiments, and delete pods
- Must have PodSecurityPolicy or PodSecurity Admission allowing privileged containers for netem/io-stress

**AWS:**
- IAM policy needs `ec2:TerminateInstances`, `autoscaling:TerminateInstanceInAutoScalingGroup`
- SSM Session Manager optional for remote command execution
- CloudWatch metrics required for blast radius calculation

**Azure:**
- Contributor role on target resource groups
- VMSS instance deletion permissions
- Azure Monitor metrics for blast radius

**GCP:**
- Compute.instanceAdmin.v1 role
- MIG (Managed Instance Group) permissions
- Cloud Monitoring metrics API enabled

**On-premises:**
- SSH access to physical/virtual hosts
- tc/netem must be available (requires NET_ADMIN capability)
- Stress-ng must be installed on target nodes (via DaemonSet)
```