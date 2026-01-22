---
title: "JVM Pod 재시작 시 초기 요청 실패, StartupProbe로 해결하기"
weight: 4
description: "Spring Boot 애플리케이션이 Pod 재시작 후 초기 요청에서 타임아웃이 발생한 문제를 StartupProbe로 해결한 경험."
tags: ["Kubernetes", "JVM", "Spring Boot", "StartupProbe", "EKS"]
keywords: ["JVM Cold Start", "StartupProbe", "ReadinessProbe", "Pod 재시작", "워밍업"]
---

Kubernetes에서 JVM 애플리케이션은 시작이 느리다. Pod가 뜨고 컨테이너가 Running 상태가 되어도, JVM이 준비되기까지는 시간이 더 걸린다. 이 간극을 무시하면 초기 요청이 실패한다.

Karpenter 노드 정리로 Pod가 재스케줄링될 때마다 초기 API 요청이 타임아웃으로 실패하는 문제가 있었다. ReadinessProbe만으로는 부족했다. StartupProbe와 전용 엔드포인트를 도입해서 해결한 과정을 공유한다.

## 문제 상황

새벽에 Karpenter가 비용 최적화를 위해 유휴 노드를 정리하면서 Pod가 다른 노드로 이동했다. Pod 재시작 자체는 정상이었지만, 재시작 직후 들어온 요청들이 실패했다.

**에러 증상:**
- 호출측에서 5초 타임아웃 설정
- Pod 시작 직후 들어온 요청이 타임아웃
- Datadog APM에 500 에러 기록

```
03:09:48  컨테이너 시작 완료
03:11:47  Healthy 상태 복구
          ↑ 이 사이 약 2분간 요청 실패
```

## 원인 분석

JVM 애플리케이션의 Cold Start 문제였다.

**JVM 시작 과정:**
1. 컨테이너 시작 → Kubernetes는 Running으로 인식
2. JVM 초기화 → 클래스 로딩, JIT 컴파일 준비
3. Spring Context 로딩 → Bean 초기화, 의존성 주입
4. DB 커넥션 풀 초기화 → 실제 연결 수립
5. **이 시점에서야 요청 처리 가능**

문제는 1~4 단계 사이에 트래픽이 들어온 것이다. 컨테이너는 Running이지만 애플리케이션은 아직 준비가 안 됐다.

기존에는 ReadinessProbe만 설정되어 있었는데, initialDelaySeconds가 10초로 짧았다. JVM 워밍업에는 턱없이 부족한 시간이었다.

## 왜 ReadinessProbe만으로 부족했나

단순히 `initialDelaySeconds`를 늘리면 되지 않을까? 문제는 **엔드포인트의 체크 범위**였다.

기존 ReadinessProbe는 `/status/health-check`를 사용했다. 이 엔드포인트는 가벼운 상태 체크만 수행한다. JVM이 뜨면 바로 200 OK를 반환할 수 있다.

하지만 실제 요청을 처리하려면 DB 커넥션 풀이 초기화되어야 한다. `/health-check`가 성공해도 DB는 아직 준비 안 됐을 수 있다.

```
JVM 시작 → /health-check 성공 → 트래픽 유입 → DB 연결 안 됨 → 타임아웃
```

**필요한 것:**
- 시작 시에는 DB 연결까지 확인하는 **무거운 체크**
- 운영 중에는 10초마다 실행되는 **가벼운 체크**

두 가지를 분리하려면 엔드포인트가 달라야 하고, 그래서 StartupProbe가 필요했다.

## 해결: StartupProbe + 전용 엔드포인트

### 전용 엔드포인트 구현

단순히 200 OK를 반환하는 것이 아니라, DB 연결까지 확인하는 엔드포인트를 만들었다.

```java
@GetMapping("/status/startup-check")
public ResponseEntity<String> startupCheck() {
    // 간단한 SELECT 쿼리로 DB 연결 확인
    jdbcTemplate.queryForObject("SELECT 1", Integer.class);
    return ResponseEntity.ok("OK");
}
```

**왜 DB 쿼리를 포함하는가:**
- JVM이 떴어도 DB 커넥션 풀이 초기화 안 됐을 수 있음
- 실제 요청 처리에 필요한 최소 조건을 검증
- SELECT 1은 가볍지만 커넥션 수립은 확인됨

### Helm 설정

```yaml
startupProbe:
  httpGet:
    path: /status/startup-check
    port: http
  initialDelaySeconds: 100
  periodSeconds: 20
  timeoutSeconds: 5
  failureThreshold: 10

readinessProbe:
  httpGet:
    path: /status/health-check
    port: http
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

**설정값 의미:**
- `initialDelaySeconds: 100` - 컨테이너 시작 후 100초 대기
- `periodSeconds: 20` - 20초마다 체크
- `failureThreshold: 10` - 10번까지 실패 허용

**최대 허용 시작 시간:**
```
100초 + (20초 × 10회) = 300초 (5분)
```

JVM + Spring Boot + DB 커넥션 초기화에 충분한 시간이다.

### 엔드포인트 분리

StartupProbe와 ReadinessProbe의 엔드포인트를 분리했다.

| Probe | 엔드포인트 | 체크 내용 |
|-------|-----------|----------|
| StartupProbe | `/status/startup-check` | DB 연결 포함, 무거운 초기화 검증 |
| ReadinessProbe | `/status/health-check` | 가벼운 상태 체크 |

StartupProbe는 시작 시 한 번만 성공하면 되므로 DB 쿼리를 포함해도 괜찮다. ReadinessProbe는 계속 실행되므로 가볍게 유지한다.

## Kubernetes Probe 동작 원리

HTTP Probe의 성공/실패 기준:

| 응답 코드 | 결과 |
|----------|------|
| 200-399 | 성공 |
| 400-499 | 실패 → 재시도 |
| 500-599 | 실패 → 재시도 |
| 타임아웃 | 실패 → 재시도 |

워밍업 중 타임아웃이 발생해도 StartupProbe가 재시도하므로, 결국 워밍업이 완료되면 성공한다.

## 결과

StartupProbe 적용 후 Pod 재시작 시 초기 요청 실패가 사라졌다.

**Before:**
- Pod 재시작 직후 2분간 요청 실패
- 호출측 타임아웃 에러 발생

**After:**
- StartupProbe 성공 전까지 트래픽 차단
- 워밍업 완료 후에만 요청 수신
- 에러 0건

## 정리

JVM 애플리케이션의 Pod 재시작 시 초기 요청 실패 문제 해결:

1. **원인 파악**: JVM Cold Start + DB 커넥션 초기화 시간
2. **엔드포인트 분리**: 시작용 무거운 체크 vs 운영용 가벼운 체크
3. **StartupProbe 도입**: DB 연결까지 확인하는 `/startup-check`
4. **충분한 시간 확보**: 최대 5분까지 워밍업 대기

핵심은 **체크 로직의 분리**다. 가벼운 health-check는 "JVM은 떴지만 DB 연결 안 됨" 상태를 잡지 못한다. 시작 시에만 무거운 체크를 수행하고, 운영 중에는 가벼운 체크를 유지하려면 StartupProbe와 ReadinessProbe를 분리해야 한다.
