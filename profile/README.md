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
