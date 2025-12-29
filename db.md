# Saleor RDS 데이터베이스 가이드

## 📊 데이터베이스 개요

| 항목 | 값 |
|------|-----|
| **엔진** | PostgreSQL 15.10 |
| **인스턴스** | db.m5.large |
| **데이터베이스명** | saleor |
| **총 테이블 수** | 143개 |

---

## 🔌 데이터베이스 접속 방법

### Django Shell 사용 (권장)

```bash
# Backend Pod에서 Django Shell 실행
kubectl exec -it deployment/backend -n kyeol-dev -- python manage.py shell
```

```python
# 사용자 조회
from saleor.account.models import User
User.objects.all()

# 주문 조회
from saleor.order.models import Order
Order.objects.all()

# 상품 조회
from saleor.product.models import Product
Product.objects.all()
```

### 직접 SQL 쿼리 실행

```python
from django.db import connection
cursor = connection.cursor()
cursor.execute("SELECT * FROM account_user")
cursor.fetchall()
```

---

## 📋 주요 테이블 카테고리

### 1. 사용자 관련 (account_*)

| 테이블 | 설명 |
|--------|------|
| `account_user` | 사용자 정보 (이메일, 이름, 스태프 여부 등) |
| `account_address` | 배송/청구 주소 |
| `account_customerevent` | 고객 활동 로그 |
| `account_group` | 사용자 그룹 |

### 2. 주문 관련 (order_*)

| 테이블 | 설명 |
|--------|------|
| `order_order` | 주문 정보 (번호, 상태, 금액 등) |
| `order_orderline` | 주문 상품 라인 |
| `order_orderevent` | 주문 이벤트 로그 |
| `order_fulfillment` | 배송 처리 정보 |

### 3. 상품 관련 (product_*)

| 테이블 | 설명 |
|--------|------|
| `product_product` | 상품 기본 정보 |
| `product_productvariant` | 상품 변형 (사이즈, 색상 등) |
| `product_producttype` | 상품 유형 |
| `product_category` | 카테고리 |
| `product_collection` | 컬렉션 |
| `product_productmedia` | 상품 이미지 |

### 4. 체크아웃 관련 (checkout_*)

| 테이블 | 설명 |
|--------|------|
| `checkout_checkout` | 장바구니/체크아웃 세션 |
| `checkout_checkoutline` | 장바구니 상품 라인 |

### 5. 채널 관련 (channel_*)

| 테이블 | 설명 |
|--------|------|
| `channel_channel` | 판매 채널 (Default Channel 등) |

### 6. 할인/프로모션 (discount_*)

| 테이블 | 설명 |
|--------|------|
| `discount_promotion` | 프로모션 |
| `discount_promotionrule` | 프로모션 규칙 |

---

## 📊 데이터 조회 예시

### 사용자 조회

```python
from saleor.account.models import User

# 모든 사용자
for u in User.objects.all():
    print(f'Email: {u.email}, Staff: {u.is_staff}')

# 스태프만 조회
User.objects.filter(is_staff=True)

# 일반 회원만 조회
User.objects.filter(is_staff=False)
```

### 주문 조회

```python
from saleor.order.models import Order

# 모든 주문
for o in Order.objects.all():
    print(f'Order #{o.number}: {o.status}, {o.total_gross_amount} {o.currency}')

# 특정 상태 주문 조회
Order.objects.filter(status='unconfirmed')

# 주문 라인 포함 조회
for o in Order.objects.prefetch_related('lines').all():
    print(f'Order #{o.number}')
    for line in o.lines.all():
        print(f'  - {line.product_name}: {line.quantity}개')
```

### 상품 조회

```python
from saleor.product.models import Product, ProductVariant

# 모든 상품
for p in Product.objects.all():
    print(f'{p.name}: {p.slug}')

# 상품 변형 조회
for v in ProductVariant.objects.all():
    print(f'{v.name}: SKU={v.sku}')
```

---

## 🔐 데이터베이스 자격증명

자격증명은 AWS Secrets Manager에 저장됩니다:

```bash
# Secret ARN 확인
cd infrastructure/terraform && terraform output rds_secret_arn

# Secret 내용 조회 (AWS CLI)
aws secretsmanager get-secret-value --secret-id $(terraform output -raw rds_secret_arn) --query SecretString --output text | jq
```

---

## 📚 참고

- [Saleor 공식 문서](https://docs.saleor.io/)
- [Django ORM 문서](https://docs.djangoproject.com/en/5.0/topics/db/queries/)
