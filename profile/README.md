# cattomato-msa

MSA + Kubernetes 실습 스터디 조직입니다.

## 스택

- **Backend** — Java Spring Boot 3, MySQL, Redis
- **Infra** — AWS EKS, ECR, ALB, Terraform
- **CI/CD** — GitHub Actions + ArgoCD

## 레포

| 레포 | 설명 |
|---|---|
| terraform | AWS 인프라 (EKS, VPC, ECR, ALB) |
| k8s-manifests | K8s 배포 정의 (ArgoCD 소스) |
| service-a/b/c | Spring Boot 마이크로서비스 |
| frontend | 프론트엔드 |

## 파이프라인

```
코드 푸시 → GitHub Actions (빌드 + ECR 푸시) → ArgoCD (EKS 자동 배포)
```

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

Terraform 상태 파일을 S3에 저장하기 위해 버킷 생성

```bash
aws s3api create-bucket \
  --bucket cattomato-msa-tfstate \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2
```

`terraform/backend.tf`에 아래 설정 적용

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

각 서비스 레포(service-a/b/c, frontend)에 아래 Secrets 등록

| Secret | 설명 |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM 액세스 키 |
| `AWS_SECRET_ACCESS_KEY` | IAM 시크릿 키 |
| `ECR_REGISTRY` | ECR 레지스트리 URL |
| `MANIFEST_TOKEN` | k8s-manifests 레포 쓰기 권한 PAT |

### 5. kubeconfig 설정

EKS 클러스터 생성 후 로컬 kubectl 연결

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
```

ArgoCD UI 접속 (포트포워딩)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080
```

### 7. ALB Controller 설치

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=cattomato-msa-cluster \
  --set serviceAccount.create=true
```
