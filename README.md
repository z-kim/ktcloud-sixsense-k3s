# K3s Security Project

본 리포지토리는 kt cloud 클라우드 인프라 2기 심화프로젝트 *Cost-Zero 기반의 오픈소스 통합 보안 운영 체계 구축* 에서 K3S부분을 프로비저닝 하기 위한 리포지토리입니다.

AWS 위에 K3s 환경을 만들고, 그 위에 ingress-nginx, Checkins 앱(임시), Falco, Node Exporter를 올리고, AWS RDB(MySQL)를 연결합니다.

다만 이 저장소의 Terraform 코드는 프로젝트의 메인 인프라 전체를 담당하기 위해 만든 것은 아닙니다. 제 담당 범위인 K3s 영역을 반복해서 올리고 내리며 검증하기 편하도록 정리한 보조 프로비저닝 구성이고, 메인 인프라 설계와 구현은 다른 팀원이 진행합니다.

- `infra/`: Terraform으로 AWS 인프라 생성
- `ansible/`: bastion, k3s server 등의 고정 노드 설정
- `cluster/`: K3s에 올라갈 Kubernetes 리소스
- `app/`: 애플리케이션 이미지 빌드용 소스
- `ops-scripts/`: HPA 반응과 Pod 복구 동작을 실습할 때 쓰는 보조 스크립트

현재 구조에서는 Terraform / Ansible 로 인프라와 K3s bootstrap 까지 준비한 뒤, control-plane 에서 아래 명령을 한 번 실행해야 웹앱 workload 가 실제로 올라갑니다.

```bash
kubectl apply -k ~/cluster/workloads/apps/checkins/overlays/dev
```

## 이 리포지토리에서 보여주고 싶은 것

이 리포지토리에서 실제로 강조하고 싶은 부분은 아래와 같습니다.

- K3s control-plane 과 worker node 로 구성된 클러스터
- ALB 를 통해 외부 요청이 worker node 의 ingress-nginx 로 유입되는 구조
- worker ASG 와 HPA 를 함께 사용한 복구 / 확장 흐름
- Falco, Falcosidekick, node-exporter 같은 보안 / 관찰성 구성
- MySQL RDS 와 연결되는 애플리케이션 구조

AWS 위에서 K3s 기반 서비스가 어떻게 올라가고 어떻게 반응하는지를 보여주는 데 더 초점을 두고 있습니다.

## 현재 구성 핵심

현재 구조는 아래처럼 이해하면 됩니다.

네트워크 흐름:
- 외부 사용자는 ALB 로 접속합니다.
- ALB 는 worker node 의 ingress-nginx NodePort 로 요청을 전달합니다.
- ingress-nginx 는 Kubernetes Service 를 통해 Checkins Pod 로 요청을 넘깁니다.

앱 구성:
- Checkins 앱은 K3s 위에서 동작합니다.
- 애플리케이션은 MySQL RDS 와 연결될 수 있습니다.
- Falco 와 Falcosidekick 이 런타임 보안 이벤트를 감시합니다.
- node-exporter 가 각 노드 메트릭을 노출합니다.

장애 대응:
- worker node 는 ASG 로 관리되어 인스턴스가 죽으면 다시 생성됩니다.
- Checkins Pod 수는 HPA 가 CPU 사용률을 기준으로 `2~4` 범위에서 조정합니다.

## K3s control-plane 과 worker 구성
### control-plane

control-plane은 private subnet 에 단일 EC2로 두었습니다. 이 인스턴스는 Ansible 로 K3s server 를 설치하고, `cluster/bootstrap/` 아래 리소스를 `/var/lib/rancher/k3s/server/manifests/` 에 복사합니다. bootstrap 을 통해 ingress-nginx, Falco, node-exporter 등이 만들어집니다.

DB Secret 값은 Ansible 로 주입합니다. 또한 k3s join token 은 Terraform 에서 생성되고, Ansible 실행 시 control-plane 에 전달되게 하였습니다.

관련 코드는 아래 파일에서 확인할 수 있습니다.

- [ansible/roles/k3s_server/tasks/main.yaml](ansible/roles/k3s_server/tasks/main.yaml)
- [infra/environments/dev/main.tf](infra/environments/dev/main.tf)

예를 들어 Terraform 에서는 control-plane 인스턴스를 private subnet 에 두고 있습니다.

```hcl
module "k3s_server" {
  subnet_id = module.vpc.private_subnet_ids[0]
}
```

Ansible 에서는 같은 노드에 K3s server 를 설치하면서 `k3s_token` 을 전달합니다.

```yaml
- name: Install k3s server
  environment:
    INSTALL_K3S_EXEC: server
    K3S_TOKEN: "{{ k3s_token }}"
```

### worker node

worker node 는 별도 Auto Scaling Group 으로 관리합니다. worker 인스턴스는 launch template 의 user data 에서 K3s agent 를 설치하고, control-plane 의 private IP 와 같은 Terraform 생성 k3s join token 을 사용해 자동 조인합니다. 또한 worker 에는 `node-role.k3s-project.io/worker=true`, `workload=general` label 을 부여해 두어서, worker 전용 workload 를 분리해 올릴 수 있게 했습니다.

이 구조를 택한 이유는 control-plane 의 초기 설정과 worker 의 증감 성격이 다르기 때문입니다. control-plane 은 상대적으로 고정된 노드라 Ansible 로 상태를 맞추기 쉽고, worker 는 ASG 로 수를 유지하거나 다시 만들기 쉽게 두었습니다.

관련 코드는 아래 파일에서 확인할 수 있습니다.

- [infra/modules/k3s_worker_group/main.tf](infra/modules/k3s_worker_group/main.tf)
- [infra/templates/k3s_worker_user_data.sh.tftpl](infra/templates/k3s_worker_user_data.sh.tftpl)

worker 쪽은 아래처럼 launch template 과 ASG 가 연결됩니다.

```hcl
user_data = base64encode(templatefile("${path.module}/../../templates/k3s_worker_user_data.sh.tftpl", {
  server_private_ip = var.server_private_ip
  k3s_token         = var.k3s_token
}))

resource "aws_autoscaling_group" "this" {
  target_group_arns = var.target_group_arns
  health_check_type = "EC2"
}
```

## ALB 와 worker group 구성

외부 요청 흐름은 `ALB -> worker node 의 ingress-nginx -> Service -> Pod` 입니다.

ALB 는 public subnet 에 배치됩니다. target group 은 worker group 을 바라보며, worker node 의 NodePort 로 트래픽을 전달합니다. 현재 ingress-nginx 는 `DaemonSet` 으로 띄우고, worker node 에만 스케줄되도록 설정했습니다. 또한 Service 타입은 `NodePort` 이며 `30080` / `30443` 포트를 사용합니다.

즉 ALB 는 Kubernetes Pod 를 직접 바라보지 않고, worker node 들의 ingress-nginx 진입점만 바라봅니다. ingress-nginx 가 다시 Ingress 규칙을 해석해서 웹앱의 Service 로 요청을 넘기고, Service 가 실제 Pod 로 라우팅합니다.

worker group 의 ASG 교체 기준은 `ELB` 가 아니라 `EC2` health check 입니다. 앱 단의 장애는 Kubernetes 의 pod 레벨에서 대응하고, 인스턴스 단의 장애는 AWS Auto Scaling 이 관리합니다. ALB 는 target health 를 기준으로 각 worker 에 트래픽을 보낼지 말지를 판단합니다.

웹앱은 K3s 안에서 HPA 로 확장합니다. 기본 replica 는 `2`, 최대 replica 는 `4` 이고 CPU 사용률 기준으로 scale-out / scale-in 하도록 두었습니다. 즉 node 레벨 복구는 ASG 가, pod 레벨 확장은 HPA 가 맡는 구조입니다.

관련 코드는 아래 파일에서 확인할 수 있습니다.

- [infra/modules/alb/main.tf](infra/modules/alb/main.tf)
- [cluster/bootstrap/ingress-nginx/values.yaml](cluster/bootstrap/ingress-nginx/values.yaml)

예를 들어 Terraform 에서는 ALB target group 을 worker 쪽으로 연결합니다.

```hcl
resource "aws_lb_target_group" "this" {
  port        = var.target_port
  target_type = "instance"

  health_check {
    path    = "/"
    matcher = "200-404"
  }
}
```

그리고 ingress-nginx 쪽은 Kubernetes 에서 아래처럼 NodePort 를 사용합니다.

```yaml
controller:
  kind: DaemonSet
  service:
    type: NodePort
    nodePorts:
      http: 30080
      https: 30443
```

## Checkins 앱

현재 Checkins 앱은 MySQL RDS와 연결되어 체크인 목록을 생성, 조회, 삭제하는 예제 웹 애플리케이션입니다.

현재 web app 이 뜨는 흐름은 아래와 같습니다.

- 사용자가 ALB DNS 로 접속합니다.
- ALB 가 worker node 의 ingress-nginx NodePort 로 요청을 보냅니다.
- ingress-nginx 가 `checkins` Service 로 요청을 넘깁니다.
- Service 가 실제 `checkins` Pod 로 트래픽을 전달합니다.

즉 외부에서는 ALB 로 접속하지만, 내부에서는 `Ingress -> Service -> Pod` 흐름으로 애플리케이션이 동작합니다.

현재 배포되는 Docker 이미지는 `app/` 디렉터리의 애플리케이션 소스와 Dockerfile 을 기준으로 빌드한 이미지를 사용하는 구조입니다.

앱을 교체하거나 새 이미지를 사용할 때도 최소한 아래 endpoint 는 유지하는 것이 좋습니다.

- `GET /health`
- `GET /ready`

앱 설정을 바꾸고 싶다면 주로 아래 YAML 을 수정합니다.

- [patch-image.yaml](cluster/workloads/apps/checkins/overlays/dev/patch-image.yaml)
  - Docker Hub 이미지 이름과 태그를 바꿉니다.
- [patch-resources.yaml](cluster/workloads/apps/checkins/overlays/dev/patch-resources.yaml)
  - CPU / 메모리 requests, limits 를 바꿉니다.
- [patch-replicas.yaml](cluster/workloads/apps/checkins/overlays/dev/patch-replicas.yaml)
  - 기본 replica 수를 바꿉니다.
- [patch-tolerations.yaml](cluster/workloads/apps/checkins/overlays/dev/patch-tolerations.yaml)
  - node 장애 시 pod 이동 관련 toleration 동작을 바꿉니다.

수정 후에는 control-plane 에서 아래 명령으로 다시 적용합니다.

```bash
kubectl apply -k ~/cluster/workloads/apps/checkins/overlays/dev
```

이때 `overlays/dev` 는 `base` 를 포함하는 Kustomize 오버레이이므로, 실제 적용 시에는 `base` 리소스도 함께 반영됩니다.

## 현재 환경 검증 포인트

이미 올라가 있는 환경을 기준으로 먼저 보고 싶은 포인트는 아래와 같습니다.

- ALB DNS 로 접속했을 때 Checkins 앱이 응답하는지 확인합니다.
- control-plane 에서 `kubectl get nodes`, `kubectl get pods -A` 가 정상 동작하는지 확인합니다.
- Checkins Pod 에 CPU 부하를 걸었을 때 HPA 가 어떻게 반응하는지, AWS 에서 worker node 를 중지하거나 종료했을 때 ASG 가 다시 worker 를 생성하는지도 확인할 수 있습니다.
- HPA, worker 장애 대응, node-exporter, Falcosidekick, ModSecurity 확인 절차는 [docs/k3s-verification-guide.md](docs/k3s-verification-guide.md) 에 정리해 두었습니다.

즉 이 저장소는 "직접 프로비저닝하는 법"보다 먼저, "현재 K3s 구성이 어떤 상태인지 어떻게 확인할 것인가"를 보는 쪽이 더 자연스럽습니다.

## 본 리포지토리를 다시 프로비저닝해보고 싶다면

재프로비저닝 절차는 [docs/reprovisioning-guide.md](docs/reprovisioning-guide.md) 에 따로 정리해 두었습니다.
