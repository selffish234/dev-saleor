지금 나는 /home/selffish234/workspace/dev-saleor/mgmt.md 이 파일을 바탕으로 /home/selffish234/workspace/dev-saleor 이 프로젝트를 진행했고 이제 다른 mgmt를 위한 다른 vpc를 만들고 싶어. 이 mgmt vpc에 대한 정보는 다음과 같아

mgmt vpc는 3tier일 필요 없이 public과 private만 있으면 되고 public은 1개의 az를 가진 서브넷으로 private도 1개의 az를 가진 서브넷으로 구성하면 될 것 같다.

mgmt엔 우선 argo cd와 github action을 위한 것이야.argo cd는 github actions를 바라보다가 github action이 CI를 완료하면 argocd를 가 그걸 k8s에 적용하는 것이다.

mgmt vpc랑 dev vpc는 서로 peering으로 연결해서 ArgoCD가 dev VPC의 EKS 클러스터에 배포할거야.

ArgoCD 외부 접근을 위한 것으로 ArgoCD UI 접근을 위해서 SSM + Port Forwarding을 사용할 예정이다.
### MGMT (2AZ )

VPC: `10.40.0.0/16`

| AZ | public | ops-private |
| --- | --- | --- |
| a | `10.40.0.0/24` | `10.40.4.0/22` |
| c | `10.40.1.0/24` | `10.40.12.0/22` |

## 네이밍

# [KYEOL] AWS Resource Naming Convention V3.0

**기본 원칙:** `kyeol-[환경]-[리소스]-[상세역할]-[식별자]`

- **환경(Env):** `prod`, `stage`, `dev`, `mgmt`
- **구분자:** 하이픈() 사용, 모두 소문자 사용

## 1. VPC (Virtual Private Cloud)

가장 큰 논리적 격리 단위입니다.

| **리소스** | **규칙 (Pattern)** | **예시 (Production)** | **설명** |
| --- | --- | --- | --- |
| **VPC** | `kyeol-[env]-vpc` | `kyeol-prod-vpc` | 메인 VPC |
| **Flow Logs** | `kyeol-[env]-vpc-flowlog` | `kyeol-prod-vpc-flowlog` | 네트워크 트래픽 로그 (ISMS-P 필수) |

---

## 2. 네트워크 (Network)

서브넷, 게이트웨이, 라우팅 테이블 등 통신을 담당하는 리소스입니다.

| **리소스** | **규칙 (Pattern)** | **예시 (Production)** | **설명** |
| --- | --- | --- | --- |
| **Subnet** | `kyeol-[env]-sub-[type]-[az]` | `kyeol-prod-sub-pub-a`
`kyeol-prod-sub-app-c` | `pub`(Public), `app`, `db`, `cache` 등으로 구분
/dev의 AZ에 있어서 public만 2개의 AZ를 가진 서브넷 2개로 하고 private 서브넷은 AZ 1개로 진행 |
| **IGW** | `kyeol-[env]-igw` | `kyeol-prod-igw` | 인터넷 게이트웨이 |
| **NAT GW** | `kyeol-[env]-nat-[az]` | `kyeol-prod-nat-a` | Private Subnet 통신용 (AZ별 생성 시) |
| **Route Table** | `kyeol-[env]-rt-[type]` | `kyeol-prod-rt-pub`
`kyeol-prod-rt-pri-a` | 라우팅 테이블 (Private은 AZ별 분리 추천) |
| **EIP** | `kyeol-[env]-eip-[usage]` | `kyeol-prod-eip-nat-a` | NAT Gateway용 고정 IP |
| **Endpoint** | `kyeol-[env]-vpce-[service]` | `kyeol-prod-vpce-ssm`
`kyeol-prod-vpce-s3` | SSM, S3용 VPC 인터페이스 엔드포인트 |

---

## 3. EC2 & Compute (EKS 포함)

서버 자원 및 접근 제어(보안 그룹) 관련 리소스입니다. Bastion 없이 SSM을 사용하므로 관련 설정을 포함합니다.

| **리소스** | **규칙 (Pattern)** | **예시 (Production)** | **설명** |
| --- | --- | --- | --- |
| **EKS Cluster** | `kyeol-[env]-eks` | `kyeol-prod-eks` | 쿠버네티스 클러스터 본체 |
| **Node Group** | `kyeol-[env]-ng-[role]` | `kyeol-prod-ng-app`
`kyeol-prod-ng-batch` | EKS 워커 노드 그룹 (EC2 집합) |
| **Launch Tpl** | `kyeol-[env]-lt-[role]` | `kyeol-prod-lt-app` | 오토스케일링용 시작 템플릿 |
| **Sec Group** | `kyeol-[env]-sg-[role]` | `kyeol-prod-sg-alb`
`kyeol-prod-sg-app` | **가장 중요.** 역할별로 보안 그룹 분리 |
| **IAM Role** | `kyeol-[env]-role-[service]` | `kyeol-prod-role-eks-node` | EC2가 가질 권한 (SSM 권한 포함 필수) |
| **ALB** | `kyeol-[env]-alb-[usage]` | `kyeol-prod-alb-ext` | 외부 접속용 로드밸런서 |

---

## 4. DB (Database & Cache)

데이터 저장소입니다. 스토리지뿐만 아니라 파라미터 그룹 등 설정 파일도 포함합니다.

| **리소스** | **규칙 (Pattern)** | **예시 (Production)** | **설명** |
| --- | --- | --- | --- |
| **Aurora Clust** | `kyeol-[env]-aurora-[engine]` | `kyeol-prod-aurora-mysql` | Aurora DB 클러스터 식별자 |
| **Aurora Inst** | `kyeol-[env]-db-[role]-[num]` | `kyeol-prod-db-writer-01`
`kyeol-prod-db-reader-01` | 실제 DB 인스턴스 |
| **Redis Clust** | `kyeol-[env]-redis-[usage]` | `kyeol-prod-redis-session` | 세션 저장소용 ElastiCache |
| **Subnet Grp** | `kyeol-[env]-sng-[service]` | `kyeol-prod-sng-rds`
`kyeol-prod-sng-redis` | DB가 배치될 서브넷 그룹 지정 |
| **Param Grp** | `kyeol-[env]-pg-[engine]` | `kyeol-prod-pg-mysql8` | DB 파라미터 그룹 (Timezone 등 설정) |

---

## 5. S3 (Storage)

S3 버킷 이름은 **전 세계에서 유일(Globally Unique)** 해야 하므로, 보통 회사명과 리전 또는 환경을 조합하여 중복을 피합니다.

| **리소스** | **규칙 (Pattern)** | **예시 (Production)** | **설명** |
| --- | --- | --- | --- |
| **Bucket** | `kyeol-[env]-s3-[usage]` | `kyeol-prod-s3-assets` | 상품 이미지, 정적 파일 저장용 |
| **Log Bucket** | `kyeol-[env]-s3-logs-[type]` | `kyeol-prod-s3-logs-alb`
`kyeol-prod-s3-logs-cloudtrail` | **ISMS-P 감사 로그 저장용** (보안 중요) |
| **Backup Bkt** | `kyeol-[env]-s3-backup` | `kyeol-prod-s3-backup-db` | DB 덤프 등 장기 보관용 |

---

## 💡 [Tip] ISMS-P 대응을 위한 태그(Tag) 전략

이름뿐만 아니라 **AWS Tag**를 의무적으로 달아야 자산 관리가 인정됩니다. Terraform 코드 레벨에서 `default_tags`로 강제하는 것이 좋습니다.

```hcl
tags = {
  Project     = "Kyeol-Migration"
  Environment = "Production"      # prod, stage, dev
  Owner       = "InfraTeam"       # 관리 주체
  Service     = "Commerce"        # 서비스명
  ManagedBy   = "Terraform"       # 관리 도구
  ISMS-P      = "In-Scope"        # 인증 범위 포함 여부 (중요!)
}
```

## 보안 그룹(Security Group) 설계 (ISMS-P: Port 22 제거)

# [KYEOL] Security Group Design V2.0 (Dev Environment)

보안 그룹은 **"필요한 최소한의 권한(Least Privilege)"**만 부여하며, 서로의 **ID(보안 그룹 ID)를 참조**하는 방식(Chained Security Group)으로 설계하여 IP 변경에 영향을 받지 않도록 구성합니다.

### 1. `kyeol-dev-sg-alb` (ALB용)

외부 트래픽을 가장 먼저 받는 관문입니다.

- **Inbound (수신):**
    - **TCP 80 (HTTP):** `0.0.0.0/0` (전 세계 허용 - HTTPS 리다이렉트용)
    - **TCP 443 (HTTPS):** `0.0.0.0/0` (전 세계 허용)
- **Outbound (송신):**
    - **TCP 8080 (WAS):** Destination `kyeol-dev-sg-app` (오직 앱 서버로만 트래픽 전달)

### 2. `kyeol-dev-sg-app` (EKS Node/App용)

실제 애플리케이션이 구동되는 서버입니다. **SSH(22) 포트는 아예 열지 않습니다.**

- **Inbound (수신):**
    - **TCP 8080 (WAS):** Source `kyeol-dev-sg-alb` (오직 ALB를 통과한 트래픽만 허용)
    - **TCP 443 (HTTPS):** Source `kyeol-dev-sg-alb` (필요 시 ALB 통신용)
    - **TCP 22 (SSH):** **없음 (규칙 전면 삭제)** -> SSM 접속 사용
- **Outbound (송신):**
    - **All Traffic:** `0.0.0.0/0`
    - *(필수 사유: SSM 에이전트 통신, 외부 API 호출, OS 패치 다운로드 등을 위해 NAT Gateway로 나가는 아웃바운드가 필요합니다.)*

### 3. `kyeol-dev-sg-rds` (RDS PostgreSQL용) **[분리됨]**

데이터베이스 전용 보안 그룹입니다.

- **Inbound (수신):**
    - **TCP 3306 (MySQL):** Source `kyeol-dev-sg-app` (오직 앱 서버에서 오는 쿼리만 허용)
    - *(옵션: 관리자 접속이 필요할 경우 `kyeol-dev-sg-admin` 등을 추가 허용)*
- **Outbound (송신):**
    - **없음** (RDS는 스스로 인터넷에 접속하지 않음)


---