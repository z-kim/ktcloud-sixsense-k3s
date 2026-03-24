# K3s 확인 가이드

이 문서는 현재 클러스터가 떠 있는 상태에서 아래 항목을 확인하는 절차를 정리한 문서입니다.

- Checkins 앱에 CPU 부하를 걸었을 때 HPA 가 어떻게 반응하는지
- worker node 장애 또는 drain 상황에서 Pod 가 어떻게 이동하는지
- node-exporter 가 각 노드에 정상적으로 올라가 있는지
- Falcosidekick 이 Falco 이벤트를 실제로 받고 있는지
- ModSecurity 가 ingress-nginx 안에서 켜져 있는지

## 1. HPA 와 장애 대응 확인

### 1.1 Deployment 기본 개념

`Deployment`는 "이 앱 Pod를 몇 개, 어떤 방식으로 계속 유지할지"를 관리하는 리소스입니다.

아주 단순하게 보면 아래처럼 이해하면 됩니다.

- `Deployment`: 원하는 상태를 선언합니다.
- `ReplicaSet`: 그 상태를 맞추는 중간 관리자입니다.
- `Pod`: 실제로 실행되는 컨테이너 묶음입니다.

즉 `Deployment/checkins` 가 replica 를 `2` 로 유지하라고 되어 있으면, Pod 하나를 직접 삭제해도 Kubernetes 는 다시 2개를 맞추려 합니다. 그래서 Pod 를 지웠는데 다시 보이는 것은 `Deployment` 의 정상 복구 동작입니다.

### 1.2 현재 Deployment 구성

현재 [deployment.yaml](../cluster/workloads/apps/checkins/base/deployment.yaml) 에서 중요한 부분은 아래와 같습니다.

- `metadata.name: checkins`
- `spec.replicas: 2`
- `selector.matchLabels.app: checkins`
- `template.metadata.labels.app: checkins`
- worker node 에만 스케줄되도록 `nodeSelector`
- `/ready`, `/health` probe 사용

즉 현재 의도는 아래와 같습니다.

- 기본 Pod 수는 2개입니다.
- worker node 에만 올립니다.
- readiness / liveness 로 상태를 확인합니다.

### 1.3 Deployment 볼 때 가장 많이 쓰는 명령

현재 상태:

```bash
kubectl get deploy -n apps
kubectl describe deploy checkins -n apps
```

Pod 와 ReplicaSet 관계 보기:

```bash
kubectl get rs -n apps
kubectl get pods -n apps -o wide
kubectl get pods -n apps -w
```

롤아웃 상태 보기:

```bash
kubectl rollout status deployment/checkins -n apps
kubectl rollout history deployment/checkins -n apps
```

Pod 상세 보기:

```bash
kubectl describe pod -n apps <pod-name>
kubectl logs -n apps <pod-name>
```

### 1.4 HPA는 Deployment의 replica를 자동으로 바꾼다

HPA 는 Pod 의 CPU / 메모리 같은 지표를 보고, 직접 Pod 를 만드는 게 아니라 대상 Deployment 의 replica 수를 조정합니다.

현재 이 저장소에서는 아래처럼 설정되어 있습니다.

- 대상: `Deployment/checkins`
- 최소 replica: `2`
- 최대 replica: `4`
- 기준 지표: CPU
- 목표값: `averageUtilization: 60`

즉 HPA 는 `checkins` Deployment 를 보면서 아래처럼 동작합니다.

- 부하가 낮으면 2개 근처 유지
- 부하가 높으면 3개, 4개까지 증가

### 1.5 현재 HPA 해석

현재 HPA 는 CPU 사용률을 `requests.cpu` 기준으로 계산합니다.

즉 HPA 의 `60%` 는 아래 뜻입니다.

- `limit` 의 60% 가 아니라
- `request` 의 60%

현재 dev overlay 기준 값은 아래와 같습니다.

- `requests.cpu: 150m`
- `limits.cpu: 250m`
- `requests.memory: 128Mi`
- `limits.memory: 256Mi`

### 1.6 HPA 반응 기준 이해

즉 Pod 하나만 stress 해도 replica 가 늘어날 수 있습니다.

### 1.7 scale-in 이 바로 일어나지 않는 이유

Kubernetes HPA 는 보통 아래 특성으로 동작합니다.

- 늘어날 때는 비교적 빠르고
- 줄어들 때는 더 보수적입니다.

기본 동작은 아래와 같습니다.

- HPA sync 주기: 약 `15초`
- scale-down stabilization window: 약 `300초`

즉 stress 를 멈췄다고 replica 가 바로 `4 -> 2` 로 떨어지지 않을 수 있습니다.

현재 `checkins` HPA 에는 별도 `behavior` 설정이 없으므로 Kubernetes 기본 동작을 따릅니다.

### 1.8 실습용 명령 예시

현재 상태 보기:

```bash
kubectl get deploy -n apps
kubectl get hpa -n apps
kubectl get pods -n apps -o wide
kubectl top pod -n apps
```

Pod 하나에 CPU 부하 주기:

```bash
bash ops-scripts/stress-pods.sh -n apps -l app=checkins -p 1 -s 60
```

또는 직접 지정:

```bash
bash ops-scripts/stress-pods.sh -n apps -s 60 checkins-abcde
```

HPA 반응 보기:

```bash
kubectl get hpa -n apps -w
kubectl get pods -n apps -w
```

Pod 자체를 지워서 Deployment 복구 보기:

```bash
kubectl delete pod -n apps <pod-name>
kubectl get pods -n apps -w
```

worker node 를 AWS 에서 중지하거나 terminate 해서 ASG 재생성 보기:

- AWS 콘솔에서 worker 인스턴스를 중지하거나 terminate 합니다.
- control-plane 에서 아래 명령으로 변화를 봅니다.

```bash
kubectl get nodes -w
kubectl get pods -n apps -o wide -w
kubectl get pods -n ingress-nginx -o wide -w
```

이 경우 아래 흐름을 기대할 수 있습니다.

- 해당 node 가 `NotReady` 또는 `Unreachable` 로 바뀝니다.
- app Pod 와 ingress-nginx Pod 가 영향을 받습니다.
- toleration 시간이 지난 뒤 replacement Pod 가 다른 worker 에서 다시 뜰 수 있습니다.
- 인스턴스 자체가 비정상으로 판단되면 ASG 가 새 worker 를 다시 생성할 수 있습니다.

## 2. Node Exporter 확인

현재 저장소 기준으로 node-exporter 는 bootstrap 리소스입니다.

- [daemonset.yaml](../cluster/bootstrap/node-exporter/daemonset.yaml)

즉 control-plane 이 부팅될 때 K3s manifests 디렉터리에 시드되고, `monitoring` namespace 에 DaemonSet 으로 올라가게 됩니다.

### 2.1 DaemonSet 이 떠 있는지 확인

먼저 control-plane 에서 아래 명령을 실행합니다.

```bash
kubectl get ds -n monitoring
kubectl get pods -n monitoring -o wide
kubectl get nodes -o wide
```

확인 포인트는 아래와 같습니다.

- `node-exporter` DaemonSet 이 보이는지 확인합니다.
- `DESIRED`, `CURRENT`, `READY` 가 노드 수와 맞는지 확인합니다.
- Pod 가 control-plane, worker 노드에 각각 올라갔는지 확인합니다.

### 2.2 노드별 메트릭 확인

현재 node-exporter 는 각 노드의 `:9100` 에서 메트릭을 노출합니다. 따라서 control-plane 에서 각 노드 private IP 를 향해 직접 확인할 수 있습니다.

```bash
curl http://<control-plane-private-ip>:9100/metrics | head
curl http://<worker-1-private-ip>:9100/metrics | head
curl http://<worker-2-private-ip>:9100/metrics | head
```

특정 메트릭 계열만 보고 싶으면 아래처럼 확인합니다.

```bash
curl http://<worker-private-ip>:9100/metrics | grep node_cpu
curl http://<worker-private-ip>:9100/metrics | grep node_memory
curl http://<worker-private-ip>:9100/metrics | grep node_filesystem
```

즉 각 노드에 SSH 로 직접 들어가지 않아도, control-plane 에서 노드 private IP 를 향해 `curl` 하는 방식으로 충분히 확인할 수 있습니다.

## 3. Falcosidekick 확인

현재 baseline 에는 Falcosidekick 이 켜져 있습니다.

- [values.yaml](../cluster/bootstrap/falco/values.yaml)

현재 의도는 아래와 같습니다.

- `falcosidekick.enabled: true`
- `falcosidekick.replicaCount: 2`
- `falcosidekick.config.debug: true`

즉 Sidekick 이 켜져 있고, 이벤트를 받으면 디버그 로그로 흐름을 확인하기 쉽게 해둔 상태입니다.

### 3.1 Falco 와 Falcosidekick Pod 상태 확인

```bash
kubectl get pods -n falco
```

기대하는 상태는 아래와 같습니다.

- `falco` DaemonSet Pod 들이 보입니다.
- `falco-falcosidekick-*` Deployment Pod 들이 따로 보입니다.
- 현재 값 기준으로는 `falco` Pod 는 `1/1`, `falcosidekick` Pod 도 각각 `1/1` 로 보입니다.

리소스 단위로 보고 싶으면 아래 명령이 더 직접적입니다.

```bash
kubectl get ds,deploy,pods -n falco
```

여기서 보통 아래처럼 이해하면 됩니다.

- `ds/falco`: 노드별 Falco 에이전트
- `deploy/falco-falcosidekick`: 이벤트를 받아 후속 output 으로 넘기는 별도 Deployment

### 3.2 Sidekick 로그 확인

터미널 1 에서 아래 명령으로 Falcosidekick 로그를 확인합니다.

```bash
kubectl logs -n falco deploy/falco-falcosidekick -f
```

replica 가 2개라면 Pod 하나를 찍어서 봐도 됩니다.

```bash
kubectl logs -n falco falco-falcosidekick-55f9487d86-gpxck -f
```

replica 전체 흐름을 같이 보려면 아래처럼 label selector 로 보는 편이 더 낫습니다.

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falcosidekick -f --prefix
```

즉 아래처럼 보면 됩니다.

- Pod 하나만 빠르게 볼 때는 `deploy/falco-falcosidekick`
- 두 replica 중 어느 쪽이 이벤트를 처리하는지까지 보려면 `-l app.kubernetes.io/name=falcosidekick --prefix`

### 3.3 Falco 로그 참고 확인

터미널 2 에서 아래 명령으로 Falco 로그를 볼 수 있습니다.

```bash
kubectl logs -n falco ds/falco -f
```

다만 Falco 쪽 로그는 상황에 따라 Falcosidekick 로그보다 덜 즉각적으로 보일 수 있으므로, 현재 확인의 중심은 `falcosidekick` 로그가 잘 뜨는지에 두는 편이 좋습니다.

### 3.4 재현 쉬운 Falco 이벤트 만들기

터미널 3 에서 아래처럼 임시 BusyBox Pod 를 만들어 `exec` 합니다.

```bash
kubectl run busybox-shell-test --image=busybox -- sleep 3600
kubectl exec -it busybox-shell-test -- sh
kubectl delete pod busybox-shell-test
```

또는 아래처럼 Kubernetes API 접근 시도를 테스트할 수도 있습니다.

```bash
kubectl run busybox-api-test --image=busybox -- sleep 3600
kubectl exec busybox-api-test -- wget -qO- --no-check-certificate https://kubernetes.default.svc
kubectl delete pod busybox-api-test
```

이런 명령은 현재 저장소 기준으로 Falco 기본 rule 테스트에 자주 쓸 수 있습니다.

### 3.5 확인 포인트

성공적으로 이어지면 보통 아래 흐름을 기대합니다.

1. Falco 쪽에서 rule 이 탐지됩니다.
2. Falcosidekick 로그에도 이벤트 처리 흔적이 보입니다.

현재는 `debug: true` 상태라, 우선은 Falcosidekick 로그에 이벤트가 잘 뜨는지 확인하는 방향으로 보면 충분합니다.

## 4. ModSecurity 확인

현재 구조에서 ModSecurity 는 별도 Pod 가 아니라 ingress-nginx 안에서 동작합니다.

- [values.yaml](../cluster/bootstrap/ingress-nginx/values.yaml)
- [ingress.yaml](../cluster/workloads/apps/checkins/base/ingress.yaml)

현재 범위에서는 ModSecurity 가 실제로 차단까지 수행하는지 깊게 검증하기보다는, ingress-nginx 안에 설정이 반영되어 켜져 있는지만 확인하면 충분합니다.

### 4.1 ingress-nginx Pod 확인

```bash
kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -o wide
kubectl get svc -n ingress-nginx
```

### 4.2 ingress 내부 nginx 설정에 ModSecurity 가 반영됐는지 확인

먼저 ingress Pod 이름을 하나 확인합니다.

```bash
kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

그 다음 아래 명령을 실행합니다.

```bash
kubectl exec -n ingress-nginx <ingress-pod-name> -- sh -c "grep -i modsecurity /etc/nginx/nginx.conf"
```

여기서 `modsecurity on;` 같은 설정이 보이면 ModSecurity 가 ingress-nginx 안에 반영된 것으로 보면 됩니다.

이 방식은 "ALB 문제인지, ingress / ModSecurity 문제인지"를 분리해서 보기 좋습니다.
