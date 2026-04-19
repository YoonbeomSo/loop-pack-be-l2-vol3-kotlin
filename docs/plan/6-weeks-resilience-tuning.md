# Resilience4j 설정 튜닝 근거 체크리스트

> 각 설정값을 "왜 이 값인가?"에 대해 테스트 결과로 근거를 남긴다.
> 처음에 설정한 값 → 테스트 결과 → 조정 여부를 기록하여 블로그에서 "판단 과정"을 보여준다.

---

## 최종 설정

```yaml
resilience4j:
  circuitbreaker:
    circuit-breaker-aspect-order: 1    # CB가 바깥 (Retry보다 먼저)
    instances:
      pgCircuit:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
        slow-call-duration-threshold: 3s
        slow-call-rate-threshold: 50
        register-health-indicator: true
  retry:
    retry-aspect-order: 2              # Retry가 안쪽 (CB 안에서 동작)
    instances:
      pgRetry:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - feign.RetryableException    # Feign이 감싼 timeout/connection 에러만 Retry
        ignore-exceptions:
          - com.loopers.support.error.CoreException
        fail-after-max-attempts: true

feign:
  connectTimeout: 1s
  readTimeout: 3s
```

---

## 1. Timeout 설정 근거

### connectTimeout: 1s

| 질문 | 테스트 방법 | 기록 |
|------|-----------|------|
| PG 정상 시 TCP 연결에 걸리는 시간은? | PG 정상 상태에서 pg_communication_logs의 elapsed 확인 | 118~502ms (응답 포함) |
| PG 중단 시 연결 실패까지 걸리는 시간은? | PG 중단 후 요청 → elapsed 확인 | 2~9ms (Connection refused 즉시 반환) |
| 1s가 적절한가? | 정상 연결 시간의 몇 배인가? | 정상 수십ms × ~20배 = **충분** |

**판단**: connectTimeout=1s 유지. 정상 연결은 수십ms이며, Connection refused는 즉시 반환되므로 1s는 충분한 여유.

### readTimeout: 3s

| 질문 | 테스트 방법 | 기록 |
|------|-----------|------|
| PG 정상 시 응답 시간은? | 성공 요청의 elapsed | 평균 ~413ms, 최대 562ms |
| PG 스펙상 요청 지연 범위는? | 100~500ms (문서 기준) | 실측과 일치 |
| 3s가 적절한가? | 최대 응답 시간 대비 여유 | 562ms × ~5.3배 = **충분** |

**판단**: readTimeout=3s 유지. 정상 응답(100~562ms)의 5~6배이며, 너무 길면 스레드 점유가 늘어남.

---

## 2. Retry 설정 근거

### max-attempts: 3

| 질문 | 테스트 방법 | 기록 |
|------|-----------|------|
| Retry 3회 시 총 소요 시간은? | PG 다운 상태에서 결제 요청 | **1.30s** |
| pg_communication_logs로 확인 | PG 다운 시 로그 건수 | **3건** (Retry 정상 작동) |
| 사용자 체감 적절한가? | 1.3s는 결제 화면에서 허용 가능 | **적절** (5~10초 이내) |

**판단**: max-attempts=3 유지. 총 소요 1.3초로 사용자 체감 한계(5~10초) 이내.

### retry-exceptions 변경: SocketTimeoutException, ConnectException → feign.RetryableException

| 질문 | 테스트 결과 | 기록 |
|------|-----------|------|
| 초기 설정(SocketTimeout, ConnectException)이 동작하는가? | **동작 안 함!** | Feign이 예외를 RetryableException으로 감싸므로 원본 타입 매칭 안 됨 |
| feign.RetryableException으로 변경 후? | PG 다운 시 로그 3건 → **Retry 정상 작동** | |

**핵심 발견**: Feign은 `ConnectException`, `SocketTimeoutException`을 `feign.RetryableException`으로 래핑해서 던진다. Resilience4j의 `retry-exceptions`에는 래핑된 타입(`feign.RetryableException`)을 지정해야 한다.

### ignore-exceptions 변경: FeignException 제거

| 질문 | 테스트 결과 | 기록 |
|------|-----------|------|
| 초기 설정(FeignException ignore)이 문제인가? | **문제!** | RetryableException extends FeignException → ignore가 retry보다 우선 → Retry 전혀 안 됨 |
| FeignException 제거 후? | PG 500 → retry-exceptions에 없으므로 Retry 안 함 (정상) | |
| PG 500 시 로그 건수 | **1건** (Retry 안 함, 즉시 실패) | |

**핵심 발견**: `ignore-exceptions`는 `retry-exceptions`보다 우선 평가된다. `RetryableException extends FeignException`이므로, FeignException을 ignore에 넣으면 모든 네트워크 에러의 Retry가 차단된다. retry-exceptions 화이트리스트만으로 500 에러의 Retry 방지가 충분하므로 FeignException은 ignore에서 제거.

### Aspect 순서 변경: CB(1) → Retry(2)

| 질문 | 테스트 결과 | 기록 |
|------|-----------|------|
| 기본 순서(Retry 바깥, CB 안쪽)가 문제인가? | **문제!** | CB fallback이 CoreException을 throw → Retry는 CoreException(ignore)만 보고 재시도 안 함 |
| CB(1)→Retry(2) 순서로 변경 후? | Retry가 PG 호출을 직접 감싸고 3회 재시도 → 실패 시 CB가 fallback 호출 | **정상** |

**핵심 발견**: CB(바깥)→Retry(안쪽)이 올바른 순서. 이렇게 해야 Retry가 원본 예외(RetryableException)를 직접 받아서 재시도하고, Retry 소진 후 CB가 최종 실패를 기록한다.

### exponential-backoff: 500ms × 2

| 질문 | 테스트 방법 | 기록 |
|------|-----------|------|
| 초기 설정에서 지수 백오프가 적용되는가? | pg_communication_logs created_at 간격 | 1→2: ~513ms, 2→3: ~525ms → **적용 안 됨** (고정 500ms) |
| 원인은? | `enable-exponential-backoff: true` 누락 | multiplier만 지정하면 무시됨 |
| 수정 후 간격 확인 | `enable-exponential-backoff: true` 추가 후 재측정 | 1→2: **~517ms** (500ms), 2→3: **~1017ms** (1000ms) → **정상 작동** |
| 총 소요 시간 변화 | 수정 전 vs 수정 후 | 1.30s → **1.78s** (backoff 증가분) |

**핵심 발견**: Resilience4j에서 exponential backoff을 사용하려면 `enable-exponential-backoff: true`를 반드시 명시해야 한다. `exponential-backoff-multiplier`만 지정하면 무시되고 고정 `wait-duration` 간격으로 동작한다.

---

## 3. CircuitBreaker 설정 근거

### sliding-window-size: 10 / minimum-number-of-calls: 5

| 질문 | 테스트 결과 | 기록 |
|------|-----------|------|
| 5건째부터 실패율 계산 시작하는가? | PG 다운 후 5건째에 failureRate 계산 시작, OPEN 전이 | **정상** |
| 정상 상태에서 오탐(서킷 열림)이 발생하는가? | PG 정상(500=40%) 상태에서 CB CLOSED 유지 | **오탐 없음** |

**판단**: sliding-window-size=10, minimum-number-of-calls=5 유지.

### failure-rate-threshold: 50%

| 질문 | 테스트 결과 | 기록 |
|------|-----------|------|
| PG 정상 시 CB 실패율은? | CLOSED 상태에서 failureRate=30~40% | PG 500(40%)이 CB에 ERROR로 카운트됨 |
| 정상에서 서킷이 열리는가? | failureRate가 50%에 도달하지 않아 **CLOSED 유지** | 정상 작동 |
| 50%가 적절한가? | PG 정상 실패율(~40%)과 threshold(50%) 사이에 10%p 여유 | **적절** |

**⚠️ 핵심 확인 완료**: PG 정상 상태(500=40%)에서 CB가 열리지 않음. 40% < 50% threshold. 여유가 10%p로 좁지만, PG 장애(ConnectException) 시 100% 실패 → 즉시 OPEN 전이되므로 문제없음.

### wait-duration-in-open-state: 10s

| 질문 | 테스트 결과 | 기록 |
|------|-----------|------|
| 10초 후 HALF_OPEN으로 전이하는가? | OPEN → 10초 대기 → 다음 요청 시 **HALF_OPEN 전이 확인** | 정상 |
| HALF_OPEN 전이는 자동인가? | 아님. 요청이 와야 전이됨 (Resilience4j 기본 동작) | |

**판단**: 10s 유지. PG 재시작 후 복구까지 충분한 시간이며 사용자 체감 한계 이내.

### permitted-number-of-calls-in-half-open-state: 3

| 질문 | 테스트 결과 | 기록 |
|------|-----------|------|
| HALF_OPEN 3건으로 복구 판단 가능한가? | PG 정상: 3건 중 2건 성공(66%) → **CLOSED 복구** | 정상 |
| HALF_OPEN 3건 모두 실패하면? | PG 다운: 3건 모두 실패 → **다시 OPEN** | 정상 |

**판단**: 3건 유지. 최소한의 샘플로 PG 복구 여부 판단 가능.

---

## 4. CB 상태 전이 검증 요약

```
CLOSED ──(failureRate≥50%)──→ OPEN ──(10s 대기)──→ HALF_OPEN
                                  ↑                      │
                                  │                 과반 성공 → CLOSED
                                  │                 과반 실패 ─┘
                                  └──────────────────────┘
```

| 전이 | 조건 | 테스트 결과 |
|------|------|-----------|
| CLOSED → OPEN | failureRate ≥ 50% (10건 중 5건 실패) | PG 다운 시 5건째 OPEN 전이 확인 |
| OPEN → HALF_OPEN | wait-duration(10s) 경과 + 요청 도착 | 10초 대기 후 요청 시 HALF_OPEN 확인 |
| HALF_OPEN → CLOSED | 3건 중 과반 성공 (≥2건) | PG 정상: 2/3 성공 → CLOSED 복구 |
| HALF_OPEN → OPEN | 3건 중 과반 실패 (≥2건) | PG 다운: 3/3 실패 → OPEN 재전환 |

---

## 5. 응답 시간 비교 (CB 효과)

| 상황 | 평균 응답 시간 | 비고 |
|------|---------------|------|
| PG 정상 (CLOSED) | ~543ms | PG 네트워크 지연(100~500ms) 포함 |
| CB OPEN | ~115ms | PG 호출 없이 즉시 fallback |
| **차이** | **~4.7배** | CB가 PG 장애 시 스레드 점유 시간 대폭 감소 |

**의미**: CB OPEN 시 PG 호출 + Retry 대기 없이 즉시 실패 반환. 장애 전파 방지 + 시스템 자원 보호.

---

## 6. Fallback 로그 구분

| 상황 | Fallback type | 로그 메시지 |
|------|--------------|------------|
| Retry 소진 (ConnectException) | `CONNECTION_REFUSED` | PG 결제 요청 fallback: type=CONNECTION_REFUSED |
| Retry 소진 (SocketTimeout) | `TIMEOUT_EXHAUSTED` | PG 결제 요청 fallback: type=TIMEOUT_EXHAUSTED |
| CB OPEN (차단) | `CIRCUIT_OPEN` | PG 결제 요청 fallback: type=CIRCUIT_OPEN |
| PG 500 (Retry 안 함) | `UNKNOWN` | PG 결제 요청 fallback: type=UNKNOWN |

**의미**: 장애 원인별 로그 구분으로 운영 시 문제 원인 즉시 파악 가능.

---

## 7. 튜닝 기록

| 설정 | 초기값 | 변경값 | 변경 근거 |
|------|--------|--------|----------|
| retry-exceptions | SocketTimeoutException, ConnectException | **feign.RetryableException** | Feign이 예외를 래핑하므로 원본 타입 매칭 안 됨 |
| ignore-exceptions | CoreException, FeignException | **CoreException만** | RetryableException extends FeignException → ignore가 retry를 차단하는 버그 |
| Aspect 순서 | 기본 (Retry 바깥, CB 안쪽) | **CB(1) → Retry(2)** | CB fallback이 CoreException으로 변환 → Retry가 원본 예외를 못 봄 |
| resilience4j 설정 위치 | prd 프로파일에만 존재 | **공통 영역으로 이동** | local 프로파일에서 설정이 로드되지 않는 버그 |
| enable-exponential-backoff | 미설정 (비활성) | **true** | multiplier만 지정하면 무시됨, 명시적 활성화 필요 |
| readTimeout | 3s | 3s (유지) | 정상 응답(~500ms)의 6배, 충분 |
| max-attempts | 3 | 3 (유지) | 총 1.78초 소요 (backoff 포함), 사용자 체감 한계 이내 |
| failure-rate-threshold | 50% | 50% (유지) | PG 정상 실패율(~40%)보다 높아 오탐 없음 |
| wait-duration-in-open-state | 10s | 10s (유지) | PG 복구 시간 + 사용자 체감 균형 |

---

## 8. 블로그 작성 시 활용 포인트

테스트 결과를 블로그에 녹일 때 아래 구조로 작성:

```
1. "처음에 이 값으로 설정한 이유" (계획 단계의 판단)
2. "실제로 테스트해보니" (기대와 다른 점, 놀라운 점)
3. "그래서 값을 조정했다/유지했다" (근거와 함께)
4. "이 경험에서 느낀 점" (설정은 문서만 보고 정할 수 없다, 눈으로 봐야 한다)
```

### 핵심 발견 스토리

1. **설정이 prd 프로파일에만 있어서 local에서 기본값으로 동작** → YAML 프로파일 구조의 함정
2. **Feign의 예외 래핑(RetryableException) 때문에 retry-exceptions가 무시됨** → 프레임워크 내부 동작 이해 필요
3. **Aspect 순서(CB→Retry)가 Retry 동작에 결정적 영향** → 어노테이션 나열 순서 ≠ 실행 순서
4. **CB의 HALF_OPEN 전이는 자동이 아님** → 요청이 와야 상태 전이

핵심 메시지: **"Resilience4j 설정은 정답이 없고, 우리 시스템의 트래픽 패턴과 외부 시스템 특성에 맞춰 테스트로 튜닝해야 한다"**
