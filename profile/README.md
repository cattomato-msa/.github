# cattomato-msa

MSA + Kubernetes 실습 스터디 조직입니다.

## 스택

- **Backend** — Java 17, Spring Boot 3, MySQL, Redis, Kafka
- **AI** — RAG 기반 약 추천 / 처방 진단
- **Infra** — AWS EKS, ECR, ALB, RDS, Terraform
- **CI/CD** — GitHub Actions + ArgoCD

## 레포

| 레포 | 설명 |
|---|---|
| backend | Spring Boot 멀티 모듈 (user / pharmacy / medicine service) |
| msa-ai | RAG 기반 약 추천 및 처방 진단 서비스 |
| terraform | AWS 인프라 (EKS, VPC, RDS, ECR, ALB) |
| k8s | K8s 배포 정의 — ArgoCD 소스 |
| frontend | 프론트엔드 |

## 아키텍처

```
[Internet]
     ↓
  AWS ALB → EKS Cluster (t3.medium × 3)
              ├── user-service     (a1, a2)
              ├── pharmacy-service (b1, b2)
              ├── medicine-service (c1, c2)
              └── ai-service
              
  Redis / Kafka / RDS (Private Subnet)
```

## 파이프라인

```
코드 푸시 → GitHub Actions (빌드 + ECR 푸시) → ArgoCD (EKS 자동 배포)
```
