# cattomato-msa

MSA + Kubernetes 실습 스터디 조직입니다.  
Java Spring Boot 멀티 모듈 기반 마이크로서비스를 AWS EKS 위에 배포하고, GitHub Actions + ArgoCD로 CI/CD 파이프라인을 구성하는 것을 목표로 합니다.

---

## 스택

- **Backend** — Java 17, Spring Boot 3, MySQL, Redis, Kafka
- **Infra** — AWS EKS, ECR, ALB, RDS, Terraform
- **CI/CD** — GitHub Actions + ArgoCD (GitOps)

---

## 레포

| 레포 | 설명 |
|---|---|
| [cattomato_msa](https://github.com/cattomato-msa/cattomato_msa) | Spring Boot 멀티 모듈 (common + 3 서비스) |
| [terraform](https://github.com/cattomato-msa/terraform) | AWS 인프라 (EKS, VPC, RDS, ECR, ALB) |
| [k8s-manifests](https://github.com/cattomato-msa/k8s-manifests) | K8s 배포 정의 — ArgoCD 소스 |
| [frontend](https://github.com/cattomato-msa/frontend) | 프론트엔드 |

---

## 서비스 구조 (멀티 모듈)

```
cattomato_msa/
├── common/            # 공통 모듈 (JWT, 예외처리, ApiResponse, BaseEntity)
├── user-service/      # 사용자, 인증, 관리자
├── pharmacy-service/  # 약국, 재고, 이력
└── medicine-service/  # 의약품, 처방전, 보건소
```

각 서비스는 독립적으로 빌드되어 별도 Docker 이미지로 ECR에 푸시됩니다.

---

## 인프라 아키텍처

```
[Internet]
     ↓
  AWS ALB (L7)
  /api/users    → user-service
  /api/pharmacy → pharmacy-service
  /api/medicine → medicine-service
     ↓
┌──────────────────────────────────────────────┐
│               EKS Cluster                    │
│                                              │
│  Node 1          Node 2          Node 3      │
│  user-a1         user-a2                     │
│  pharmacy-b1                   pharmacy-b2   │
│                  medicine-c1   medicine-c2   │
│                                              │
│  (Pod Anti-Affinity — 같은 서비스 레플리카는  │
│   다른 노드에 강제 분산)                      │
│                                              │
│  Redis (공유 상태)    Kafka (이벤트)          │
└──────────────────────────────────────────────┘
     ↓ (VPC Private Subnet)
  AWS RDS (MySQL 1개, 스키마 3개)
  ├── yaktong_user      ← user-service 전용 계정
  ├── yaktong_pharmacy  ← pharmacy-service 전용 계정
  └── yaktong_medicine  ← medicine-service 전용 계정
```

### 노드 스펙

| 항목 | 스펙 |
|---|---|
| 리전 | ap-northeast-2 (Seoul) |
| EKS Node | t3.medium × 3 |
| EBS | 15GB gp3 (노드당) |
| Ingress | AWS ALB (L7) |

---

## VPC 구조

```
VPC
├── Public Subnet   → ALB (인터넷 접근 허용)
└── Private Subnet  → EKS Nodes, RDS, Redis, Kafka
                      (인터넷 직접 접근 차단)
```

RDS, Redis, Kafka는 Private Subnet에만 위치하며 같은 VPC 내 EKS 파드에서만 접근 가능합니다.  
Security Group으로 포트/소스 단위 접근을 추가 제한합니다.

---

## CI/CD 파이프라인

```
코드 푸시 (cattomato_msa)
    ↓
GitHub Actions
    ├── 빌드 + 테스트
    ├── Docker 이미지 빌드 (서비스별)
    ├── ECR 푸시 (image:git-sha)
    └── k8s-manifests 이미지 태그 업데이트
         ↓
    ArgoCD (EKS 내부)
         └── 변경 감지 → Rolling Update 자동 배포
```

---

## 서비스 간 통신 전략

서비스 간 직접 DB 접근 및 크로스 DB JOIN은 금지합니다.  
아래 3가지 전략으로 처리합니다.

| 상황 | 전략 |
|---|---|
| 실시간 데이터 필요 | API Composition — HTTP 호출 + Circuit Breaker |
| 자주 참조하는 타 서비스 데이터 | Event Replication — Kafka로 필요한 컬럼만 자기 DB에 복사 |
| 여러 서비스 걸친 트랜잭션 | Saga Pattern — 이벤트 체인 + 보상 트랜잭션 |

### 공유 상태

| 저장소 | 역할 |
|---|---|
| MySQL (서비스별 전용 DB) | 비즈니스 데이터 영구 저장 |
| Redis (전 서비스 공유) | RefreshToken, UUID 멱등성 키, 캐시 |
| Kafka | 서비스 간 비동기 이벤트, Saga 보상 트랜잭션 |

---

## K8s 핵심 설정

### Pod Anti-Affinity (노드 분산)
같은 서비스의 레플리카가 항상 다른 노드에 배치되도록 강제합니다.  
노드 하나가 죽어도 서비스가 유지됩니다.

### Stateless Pod + Shared State
파드 자체에는 상태를 저장하지 않습니다.  
모든 상태는 RDS / Redis에 저장하여 어느 레플리카가 요청을 처리해도 동일한 결과를 보장합니다.

### UUID 멱등성
요청마다 UUID를 부여하고 Redis에서 중복 체크하여 파드 재라우팅 시 중복 처리를 방지합니다.

### Resource Limits + HPA
파드별 메모리 상한을 설정하여 OOM으로 인한 노드 장애를 방지합니다.  
트래픽 증가 시 HPA가 자동으로 레플리카를 증가시킵니다.

---

## 사전 설정

### 1. 로컬 툴 설치

```bash
brew install terraform awscli kubectl helm k3d
```

### 2. AWS IAM 설정

Terraform 및 GitHub Actions에서 사용할 IAM 사용자 생성 후 아래 정책 연결

| 정책 | 용도 |
|---|---|
| AmazonEKSClusterPolicy | EKS 클러스터 관리 |
| AmazonEC2ContainerRegistryFullAccess | ECR 이미지 푸시/풀 |
| AmazonVPCFullAccess | VPC 프로비저닝 |
| ElasticLoadBalancingFullAccess | ALB 생성 |

```bash
aws configure
# AWS Access Key ID, Secret Access Key, Region(ap-northeast-2) 입력
```

### 3. Terraform S3 Backend 설정

```bash
aws s3api create-bucket \
  --bucket cattomato-msa-tfstate \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2
```

```hcl
terraform {
  backend "s3" {
    bucket = "cattomato-msa-tfstate"
    key    = "eks/terraform.tfstate"
    region = "ap-northeast-2"
  }
}
```

### 4. GitHub Secrets 설정

| Secret | 설명 |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM 액세스 키 |
| `AWS_SECRET_ACCESS_KEY` | IAM 시크릿 키 |
| `ECR_REGISTRY` | ECR 레지스트리 URL |
| `MANIFEST_TOKEN` | k8s-manifests 레포 쓰기 권한 PAT |

### 5. kubeconfig 설정

```bash
aws eks update-kubeconfig \
  --name cattomato-msa-cluster \
  --region ap-northeast-2
```

### 6. ArgoCD 설치

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 초기 admin 비밀번호 확인
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# UI 접속 (포트포워딩)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 7. ALB Controller 설치

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=cattomato-msa-cluster \
  --set serviceAccount.create=true
```

### 8. Kafka 설치

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install kafka bitnami/kafka \
  --set replicaCount=1 \
  --set kraft.enabled=true
```
