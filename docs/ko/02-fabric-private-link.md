# 02. Fabric Private Link 아키텍처

# 개요

Microsoft Fabric의 Private Networking은 크게 두 가지 아키텍처로 구분됩니다.

* Workspace-Level Private Link
* Tenant-Level Private Link

두 방식 모두 Azure Private Link 기술을 기반으로 하지만, 목적과 적용 대상은 완전히 다릅니다.

엔터프라이즈 환경에서 Microsoft Fabric를 설계할 때 이 차이를 이해하는 것은 매우 중요합니다.

일반적으로 기업 보안 환경에서는 다음과 같은 제약사항이 존재합니다.

* Public Internet 사용 제한
* Outbound All Deny 정책
* 제한된 DNS Resolution
* 엄격한 Firewall 정책
* Zero-Trust 보안 아키텍처

이러한 환경에서는 Microsoft Fabric와의 통신을 Public Endpoint가 아닌 Private Routing 기반으로 설계해야 합니다.

---

# Private Link가 필요한 이유

Microsoft Fabric는 SaaS 서비스이지만, 엔터프라이즈 환경에서는 Public Internet을 통한 직접 접근이 허용되지 않는 경우가 많습니다.

이때 자주 발생하는 오해가 있습니다.

> Fabric Workspace에 Private Link를 구성하면 모든 Fabric 통신이 가능하다.

이는 사실이 아닙니다.

Microsoft Fabric 네트워크 구조는 크게 두 개의 영역으로 나누어 이해해야 합니다.

* Data Plane(실제 데이터가 오가는 경로)
* Control Plane(리소스를 관리/설정/제어 하는 영역)

각 영역은 서로 다른 네트워크 경로를 사용합니다.

---

# Data Plane

Data Plane은 실제 데이터 접근을 담당하는 영역입니다.

대표적인 예시는 다음과 같습니다.

* Fabric Warehouse 접근
* Lakehouse 접근
* SQL Analytics Endpoint
* Query 수행

아키텍처:

```text
AKS
 ↓
Private Endpoint
 ↓
Workspace-Level Private Link
 ↓
Fabric Workspace
 ↓
Warehouse / Lakehouse
```

주요 목적:

* 데이터 조회
* Warehouse 접근
* Private SQL 통신

---

# Control Plane

Control Plane은 관리 및 운영 기능을 담당하는 영역입니다.

대표적인 예시는 다음과 같습니다.

* Capacity Metrics 조회
* ExecuteQueries API 호출
* Power BI REST API 호출
* Fabric Management API 호출
* Capacity SKU Resize

아키텍처:

```text
AKS
 ↓
Private Endpoint / Approved Outbound
 ↓
Tenant-Level Private Link
 ↓
api.powerbi.com
 ↓
Metrics / Fabric API
```

주요 목적:

* Capacity 모니터링
* Capacity 관리
* AutoScale 제어
* 관리 작업 수행

---

# Workspace-Level Private Link

## 목적

Workspace-Level Private Link는 Fabric 데이터 엔드포인트에 대한 Private Connectivity를 제공합니다.

대표적인 대상:

* Fabric Warehouse
* Lakehouse
* SQL Analytics Endpoint

즉, Data Plane 통신을 보호하기 위한 구조입니다.

---

## 아키텍처

```text
AKS / VM / On-Prem
      ↓
Private Endpoint
      ↓
Private DNS
      ↓
Workspace-Level Private Link
      ↓
Fabric Workspace
      ↓
Warehouse / Lakehouse
```

---

## 필수 구성 요소

성공적인 구성을 위해 일반적으로 다음 요소가 필요합니다.

### Fabric 설정

* Fabric Workspace
* Workspace-Level Private Link 활성화

### Azure Network

* Private Endpoint
* Private DNS Zone
* Virtual Network
* Subnet

### Security Controls

* NSG
* Firewall
* Route Table (선택)

---

## 주요 장애 요인

자주 발생하는 문제는 다음과 같습니다.

* DNS Resolution 실패
* SQL Port Timeout (1433)
* Private Endpoint 연결 성공 후 통신 실패
* NSG Outbound 차단
* Firewall 정책 차단

예시:

* Warehouse 접속 Timeout
* SQL Client 연결 실패
* AKS Pod에서 Warehouse 접근 실패

---

# Tenant-Level Private Link

## 목적

Tenant-Level Private Link는 Fabric 및 Power BI의 Control Plane API에 대한 Private Connectivity를 제공합니다.

대표적인 대상:

* Capacity Metrics
* Power BI API
* ExecuteQueries API
* Fabric Management API

즉, Control Plane 통신을 보호하기 위한 구조입니다.

---

## 아키텍처

```text
AKS / VM / On-Prem
      ↓
Private Endpoint
      ↓
Private DNS
      ↓
Tenant-Level Private Link
      ↓
api.powerbi.com
```

---

## 필수 구성 요소

### Fabric Administration

* Fabric Admin 권한
* Tenant Settings 구성

### Azure Network

* Private Link Service
* Private Endpoint
* Private DNS

### Security Controls

* Firewall
* NSG
* DNS Forwarding

---

## 주요 장애 요인

자주 발생하는 문제는 다음과 같습니다.

* api.powerbi.com 접근 불가
* ExecuteQueries API 403
* Capacity Metrics 접근 실패
* Tenant Setting 오설정

예시:

* Metrics Dataset 조회 실패
* API Authorization 실패
* DNS Resolution 오류

---

# Workspace-Level vs Tenant-Level

| 구분     | Workspace-Level       | Tenant-Level         |
| ------ | --------------------- | -------------------- |
| Scope  | Workspace             | Tenant               |
| 목적     | Data Access           | API Access           |
| Domain | Data Plane            | Control Plane        |
| 대상     | Warehouse / Lakehouse | api.powerbi.com      |
| 주요 용도  | Query / Data Access   | Metrics / Management |

---

# 핵심 아키텍처 차이

가장 중요한 아키텍처 포인트는 다음과 같습니다.

Workspace-Level Private Link와 Tenant-Level Private Link는 서로 다른 문제를 해결합니다.

Workspace-Level Private Link의 목적:

* Warehouse 접근
* Lakehouse 접근
* 데이터 조회

Tenant-Level Private Link의 목적:

* Capacity Metrics 조회
* API 호출
* Fabric 관리

이 차이를 이해하지 못하면 AutoScale 아키텍처 설계가 매우 어려워집니다.

---

# 가장 흔한 오해

가장 흔한 실수는 다음과 같습니다.

```text
Workspace-Level Private Link 구성 완료
→ Capacity Metrics도 동작할 것이다
```

실제는 다릅니다.

```text
Workspace-Level Private Link
≠
Power BI API Access
```

즉,

Warehouse 접근이 정상이어도 Capacity Metrics 조회는 실패할 수 있습니다.

그 이유는 다음과 같습니다.

* Warehouse 트래픽은 Data Plane
* Metrics/API 트래픽은 Control Plane

두 경로는 완전히 독립적입니다.

---

# Lessons Learned

Fabric AutoScale 구현에서 가장 어려운 부분은 AutoScale 로직 구현 자체가 아니었습니다.

실제 난이도는 아래 항목을 모두 이해해야 발생합니다.

* Private Networking
* Fabric Admin 설정
* DNS Routing
* Azure IAM
* API Access Path

주요 난관:

* Data Plane / Control Plane 구분
* Workspace / Tenant Scope 구분
* DNS 구성
* Enterprise Security 정책 충족

결국 성공적인 구현을 위해서는 아래 항목이 모두 맞아야 합니다.

* Networking
* Identity
* Fabric Permission
* Azure Permission
* Security Policy

하나라도 누락되면 통신이 실패할 수 있습니다.

---

# 다음 문서

* 03-capacity-metrics.md
