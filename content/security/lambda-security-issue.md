---
title: "AWS 미사용 리전에 숨어드는 공격, IAM 키 탈취의 위험성"
weight: 7
description: "탈취된 IAM 키로 미사용 리전에 Lambda, EC2를 생성해 크립토마이닝이나 권한 탈취를 시도하는 공격 패턴과 대응 방법."
tags: ["AWS", "보안", "Lambda", "IAM", "CloudTrail"]
keywords: ["AWS 보안", "IAM 키 탈취", "미사용 리전 공격", "크립토마이닝", "CloudTrail"]
---

AWS IAM 키가 탈취되면 어떤 일이 벌어질까? 공격자는 평소 사용하지 않는 리전에 리소스를 생성해서 숨는다. AWS 리전이 30개가 넘는데, 대부분 1~2개 리전만 사용한다. 나머지 리전은 모니터링 사각지대다.

## 공격 패턴

### 1단계: IAM 키 탈취

공격자가 IAM Access Key를 얻는 방법:
- GitHub에 실수로 커밋된 키
- 피싱으로 탈취
- EC2 인스턴스 메타데이터 서비스(IMDS) 악용
- 취약한 애플리케이션에서 유출

### 2단계: 미사용 리전에 리소스 생성

```mermaid
flowchart LR
    A[IAM 키 탈취] --> B[권한 테스트]
    B --> C[미사용 리전 선택]
    C --> D[Lambda/EC2 생성]
    D --> E[크립토마이닝<br/>또는 권한 탈취]

    style A fill:#dc2626,color:#fff
    style D fill:#dc2626,color:#fff
    style E fill:#dc2626,color:#fff
```

**왜 미사용 리전인가?**
- AWS 리전이 30개 이상
- 대부분 서울(ap-northeast-2) 정도만 사용
- 도쿄(ap-northeast-1), 오레곤(us-west-2) 등은 안 봄
- **모니터링 사각지대**

실제 사례에서 공격자는 `ap-northeast-1`(도쿄)에 50개 이상의 EC2 인스턴스를 생성해서 한 달간 크립토마이닝을 돌렸다. 피해자는 청구서를 보고서야 알았다. ([The Danger of Unused AWS Regions - CloudSploit](https://medium.com/cloudsploit/the-danger-of-unused-aws-regions-af0bf1b878fc))

### 3단계: 지속성 확보

공격자는 들키지 않고 오래 머물기 위해:

**Lambda 활용:**
```
- Lambda 함수 생성 (인증 없이 호출 가능하게)
- Function URL로 외부 접근 경로 확보
- EventBridge로 주기적 실행 설정
```

**IAM 백도어:**
```
- 새 IAM 사용자 생성
- 기존 Role에 악성 Trust Policy 추가
- 외부 AWS 계정에서 AssumeRole 가능하게
```

### 공격 속도

2025년 11월에 탐지된 대규모 캠페인에서: ([AWS Security Blog](https://aws.amazon.com/blogs/security/cryptomining-campaign-targeting-amazon-ec2-and-amazon-ecs/), [The Hacker News](https://thehackernews.com/2025/12/compromised-iam-credentials-power-large.html))
- 최초 접근 후 **10분 내** 크립토마이너 가동
- 한 번에 **999개 EC2 인스턴스** 생성
- **50개 이상의 ECS 클러스터** 생성

## 왜 찾기 어려운가

### CloudTrail 90일 한계

AWS CloudTrail Event History는 **90일**만 보관된다.

| 경과 | 상태 |
|------|------|
| 0~89일 | 로그 조회 가능 |
| **90일** | 로그 삭제 시작 |
| 91일~ | 추적 불가 |

**문제:**
- 90일 전에 Trail 설정을 안 했으면 로그가 없음
- 공격자가 누구 권한으로 뭘 했는지 추적 불가
- 사후에 Trail 설정해도 과거 로그는 복구 안 됨

| 설정 | 보관 기간 |
|------|----------|
| Event History (기본) | 90일 |
| Trail → S3 | 무제한 (S3 수명주기 정책 따름) |
| CloudTrail Lake | 최대 7~10년 |

**각 옵션 설명:**
- **Event History**: 자동으로 켜져있음, 무료, 90일 이상 보관 불가
- **Trail → S3**: 직접 설정 필요, S3 버킷에 로그 파일 저장, 가장 많이 사용하는 방법
- **CloudTrail Lake**: SQL로 쿼리 가능한 데이터 레이크, 비용 발생

90일 이상 보관하려면 **사고 터지기 전에 Trail을 미리 설정**해야 한다.

### 리전별 모니터링 부재

Event History는 **모든 리전에서 자동 활성화**되지만, **각 리전의 로그는 해당 리전에서만** 볼 수 있다.

Trail을 생성할 때는 기본값이 **현재 리전만** 기록이다.

```
❌ 서울 리전에서만 Trail 설정
   → 서울 로그만 S3에 저장
   → 도쿄에서 EC2 생성해도 S3에 안 남음

✓ 모든 리전 Trail 설정 필요
   → AWS 콘솔에서 "Apply trail to all regions" 활성화
   → 모든 리전 로그가 하나의 S3에 저장
```

**정리:**
| 설정 | 범위 | 보관 기간 |
|------|------|----------|
| Event History | 각 리전별 자동 활성화 | 90일 (변경 불가) |
| Trail (기본) | 현재 리전만 | S3 정책 따름 |
| Trail (All regions) | 모든 리전 | S3 정책 따름 |

## 예방: IAM 키 없애기

가장 좋은 방법은 **IAM 키 자체를 없애는 것**이다.

### 개인 IAM 계정의 문제

개발자마다 IAM 계정이 있으면:
- Access Key가 GitHub에 실수로 커밋됨
- 퇴사자 키 회수 누락
- 누가 어떤 키를 가지고 있는지 파악 어려움
- 키 유출 시 추적 어려움

### SSO 도입

**SSO**(Single Sign-On)를 도입하면 개인 IAM 키가 필요 없다.

```mermaid
flowchart LR
    subgraph Before["Before: 개인 IAM"]
        Dev1[개발자 A] --> Key1[Access Key]
        Dev2[개발자 B] --> Key2[Access Key]
        Dev3[개발자 C] --> Key3[Access Key]
    end

    subgraph After["After: SSO"]
        Dev4[개발자] --> SSO[SSO 로그인]
        SSO --> Temp[임시 자격 증명]
    end

    style Key1 fill:#dc2626,color:#fff
    style Key2 fill:#dc2626,color:#fff
    style Key3 fill:#dc2626,color:#fff
    style Temp fill:#16a34a,color:#fff
```

| 방식 | 키 유형 | 유출 위험 |
|------|--------|----------|
| 개인 IAM | 장기 Access Key | 높음 |
| **SSO** | 임시 토큰 (만료됨) | 낮음 |

**AWS IAM Identity Center**(구 AWS SSO)를 사용하면:
- 개인 Access Key 불필요
- 임시 자격 증명만 사용 (몇 시간 후 만료)
- 퇴사 시 SSO 계정 비활성화로 끝
- 모든 접근 기록이 중앙에서 관리됨

### 키가 필요한 경우

CI/CD 등 어쩔 수 없이 키가 필요하면:
- **IAM Role** 사용 (EC2, Lambda 등에 Role 부여)
- 키 사용 시 **90일마다 교체**
- **최소 권한 원칙** 적용
- GitHub Actions는 **OIDC**로 키 없이 연동 가능

## 탐지 및 대응

### 1. 미사용 리전 비활성화

AWS는 신규 리전을 기본 비활성화로 제공한다 (ap-east-1 홍콩부터).

**추가로 할 수 있는 것:**
- IAM에서 미사용 리전의 STS 토큰 발급 비활성화
- SCP(Service Control Policy)로 특정 리전 차단

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["ap-northeast-2", "us-east-1"]
        }
      }
    }
  ]
}
```

### 2. 모든 리전 CloudTrail 설정

```
AWS 콘솔 → CloudTrail → Trails → Create trail
→ "Apply trail to all regions" 체크
→ S3 버킷에 저장 (장기 보관)
```

### 3. GuardDuty 활성화

GuardDuty는 의심스러운 활동을 자동 탐지한다:
- 미사용 리전에서 API 호출
- 비정상적인 EC2/Lambda 생성
- 크립토마이닝 패턴

**모든 리전**에서 GuardDuty를 활성화해야 한다.

### 4. 알람 설정

CloudWatch 알람으로 이상 징후 감지:

```
모니터링할 API:
- ec2:RunInstances (특히 미사용 리전)
- ecs:CreateCluster
- lambda:CreateFunction
- iam:CreateUser
- iam:CreateRole
- iam:AttachRolePolicy
```

## 체크리스트

**예방:**
| 항목 | 확인 |
|------|------|
| SSO 도입 (개인 IAM 키 제거) | ☐ |
| 장기 Access Key 사용 금지 | ☐ |
| MFA 필수 설정 | ☐ |
| 미사용 리전 SCP로 차단 | ☐ |

**탐지:**
| 항목 | 확인 |
|------|------|
| CloudTrail 모든 리전 활성화 | ☐ |
| CloudTrail 로그 S3 장기 보관 | ☐ |
| GuardDuty 모든 리전 활성화 | ☐ |
| 이상 API 호출 알람 설정 | ☐ |

## 정리

AWS IAM 키가 탈취되면:

1. **미사용 리전**에 리소스를 생성해서 숨김
2. **CloudTrail 90일 한계**로 늦게 발견하면 추적 불가
3. **10분 내**에 크립토마이너 가동 가능

**예방이 최선:**
- **SSO 도입**으로 개인 IAM 키 자체를 없애기
- 장기 Access Key 대신 **임시 자격 증명** 사용

**탐지 체계:**
- CloudTrail **모든 리전** 설정 + S3 장기 보관
- GuardDuty **모든 리전** 활성화
- 미사용 리전 **SCP로 차단**

키가 없으면 탈취당할 키도 없다.
