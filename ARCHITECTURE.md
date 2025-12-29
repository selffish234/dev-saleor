# Saleor E-commerce AWS 인프라 정의서

## 1. 프로젝트 개요 및 목표

* **목표:** AWS 인프라 기반의 Saleor E-commerce 사이트 구축
* **구축 방식:** IaC (Terraform) 활용
* **현재 환경:** `dev` (구축 완료)
* **프로젝트명:** `saleor-joon`

### 소스코드 정보

| 항목 | 값 |
|------|-----|
| **Repository** | https://github.com/selffish234/dev-saleor |
| **로컬 경로** | `/home/selffish234/workspace/dev-saleor` |
| **작업 환경** | WSL2 + Ubuntu |
| **AWS 리전** | 서울 (ap-northeast-2) |

### 구성 요소

| 컴포넌트 | 디렉토리 | 원본 |
|----------|----------|------|
| **Backend** | `source/saleor` | [saleor/saleor](https://github.com/saleor/saleor) |
| **Dashboard** | `source/saleor-dashboard` | [saleor/saleor-dashboard](https://github.com/saleor/saleor-dashboard) |
| **Storefront** | `source/storefront` | [saleor/storefront](https://github.com/saleor/storefront) |

---

## 2. 아키텍처 구성 (3-Tier)

```
┌─────────────────────────────────────────────────────────────────┐
│                         CloudFront                               │
│              (dev.selffish234.cloud)                            │
└─────────────┬──────────────┬──────────────┬────────────────────┘
              │              │              │
       /dashboard/*    /graphql/*     /   (Storefront)
       /media/*        /  (API)
              │              │              │
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │   S3     │   │   ALB    │   │   ALB    │
       │ (Static) │   │          │   │          │
       └──────────┘   └────┬─────┘   └────┬─────┘
                           │              │
                    ┌──────▼──────────────▼──────┐
                    │         EKS Cluster         │
                    │  ┌─────────┐  ┌──────────┐ │
                    │  │ Backend │  │Storefront│ │
                    │  │   Pod   │  │   Pod    │ │
                    │  └────┬────┘  └──────────┘ │
                    └───────│────────────────────┘
                            │
                    ┌───────▼───────┐
                    │      RDS      │
                    │  PostgreSQL   │
                    └───────────────┘
```

### 2.1. Frontend (Static)

| 항목 | 구성 |
|------|------|
| **서비스** | CloudFront + S3 |
| **역할** | Dashboard 정적 파일, 미디어 파일 서빙 |
| **위치** | Global Service (VPC 외부) |
| **버킷** | `saleor-joon-dev-s3-static`, `saleor-joon-dev-s3-media` |

### 2.2. Load Balancer (Ingress)

| 항목 | 구성 |
|------|------|
| **서비스** | ALB (Application Load Balancer) |
| **위치** | Public Subnet |
| **역할** | API/Storefront 요청을 EKS로 전달 |

### 2.3. Backend (Compute)

| 항목 | 구성 |
|------|------|
| **서비스** | EKS Cluster (Kubernetes 1.33) |
| **위치** | Private Subnet |
| **노드 OS** | Amazon Linux 2023 |
| **인스턴스** | t3.large × 2 |
| **접속** | SSM (Bastion Host 없음) |

### 2.4. Database (Persistence)

| 항목 | 구성 |
|------|------|
| **서비스** | RDS PostgreSQL 15.10 |
| **위치** | Private Subnet |
| **인스턴스** | db.m5.large |
| **데이터베이스** | saleor |

---

## 3. 네트워크 구성 (DEV VPC)

**CIDR:** `10.10.0.0/16`

| AZ | 서브넷 역할 | CIDR | 용도 |
|----|-------------|------|------|
| **a** | Public | `10.10.0.0/24` | NAT Gateway, ALB |
| **a** | App-Private | `10.10.4.0/22` | EKS Worker Nodes |
| **a** | Data-Private | `10.10.9.0/24` | RDS |
| **c** | Public | `10.10.1.0/24` | ALB (고가용성) |
| **c** | Data-Private | `10.10.10.0/24` | RDS Subnet Group용 |

---

## 4. 리소스 네이밍 컨벤션

**패턴:** `saleor-joon-[env]-[resource]-[detail]`

| 카테고리 | 패턴 | 예시 |
|----------|------|------|
| VPC | `saleor-joon-[env]-vpc` | `saleor-joon-dev-vpc` |
| Subnet | `saleor-joon-[env]-sub-[type]-[az]` | `saleor-joon-dev-sub-pub-a` |
| EKS | `saleor-joon-[env]-eks` | `saleor-joon-dev-eks` |
| Node Group | `saleor-joon-[env]-ng-[role]` | `saleor-joon-dev-ng-app` |
| ALB | `saleor-joon-[env]-alb-[usage]` | `saleor-joon-dev-alb-ext` |
| RDS | `saleor-joon-[env]-db-[role]-[num]` | `saleor-joon-dev-db-writer-01` |
| S3 | `saleor-joon-[env]-s3-[usage]` | `saleor-joon-dev-s3-static` |
| ECR | `saleor-joon-[env]-[app]` | `saleor-joon-dev-backend` |

---

## 5. 태깅 전략

```hcl
default_tags {
  tags = {
    Project     = "Kyeol-Migration"
    Environment = "dev"
    Owner       = "InfraTeam"
    Service     = "Commerce"
    ManagedBy   = "Terraform"
    ISMS-P      = "In-Scope"
  }
}
```

---

## 6. 보안 그룹 설계

### 6.1. ALB용 (`saleor-joon-dev-sg-alb`)

| Direction | Port | Source/Dest |
|-----------|------|-------------|
| Inbound | 80 (HTTP) | `0.0.0.0/0` |
| Inbound | 443 (HTTPS) | `0.0.0.0/0` |
| Outbound | 8000 | `sg-app` |
| Outbound | 3000 | `sg-app` |

### 6.2. App/EKS Node용 (`saleor-joon-dev-sg-app`)

| Direction | Port | Source/Dest |
|-----------|------|-------------|
| Inbound | 8000 (Backend) | `sg-alb` |
| Inbound | 3000 (Storefront) | `sg-alb` |
| Outbound | All | `0.0.0.0/0` (NAT 통신) |

### 6.3. RDS용 (`saleor-joon-dev-sg-rds`)

| Direction | Port | Source/Dest |
|-----------|------|-------------|
| Inbound | 5432 (PostgreSQL) | `sg-app` |
| Outbound | - | 없음 |

---

## 7. 배포 시 주의사항

### 7.1. Storefront Docker 빌드
- `NEXT_PUBLIC_SALEOR_API_URL` 빌드 인자 필수
- GraphQL Codegen: 로컬 스키마 파일 사용 (`GITHUB_ACTION=generate-schema-from-file`)

### 7.2. Dashboard 빌드
- **pnpm 사용** (npm 아님)
- **CI=true** 설정 필수 (husky 에러 방지)
- Node.js v20 또는 v22 필요

### 7.3. Kubernetes 설정
- `ALLOWED_CLIENT_HOSTS` 환경변수 필수
- Migration Job에 `RSA_PRIVATE_KEY` 환경변수 필요

### 7.4. S3 미디어 스토리지
- Backend에 IRSA (IAM Role for Service Account) 설정
- `DEFAULT_FILE_STORAGE=saleor.core.storages.S3MediaStorage`

---

## 8. 주요 URL

| 서비스 | URL |
|--------|-----|
| **Storefront** | https://dev.selffish234.cloud |
| **Dashboard** | https://dev.selffish234.cloud/dashboard/ |
| **GraphQL API** | https://dev.selffish234.cloud/graphql/ |

---

## 9. 관련 문서

| 문서 | 설명 |
|------|------|
| [QUICK_START.md](./QUICK_START.md) | 빠른 시작 가이드 |
| [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) | 트러블슈팅 기록 |
| [DASHBOARD_GUIDE.md](./DASHBOARD_GUIDE.md) | Dashboard 사용 가이드 |
| [db.md](./db.md) | 데이터베이스 가이드 |
