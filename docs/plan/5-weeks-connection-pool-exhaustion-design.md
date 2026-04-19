# Connection Pool Exhaustion 증명 테스트 설계

## 배경

비관적 락은 `@Transactional` 블록이 끝날 때까지 DB 커넥션을 점유한다.
동시 요청이 커넥션 풀 사이즈를 초과하면 connection timeout이 발생하며, 이는 비관적 락의 구조적 한계다.

## 목적

비관적 락 + 동시 주문이 커넥션 풀을 고갈시키는 현상을 테스트 코드로 증명한다.
실패 원인이 "비즈니스 로직(재고 부족)"이 아니라 "인프라 한계(커넥션 획득 실패)"임을 명확히 한다.

## 테스트 환경

- 커넥션 풀 사이즈: 10 (test 프로파일)
- connection-timeout: 3초
- Testcontainers MySQL 8.0

## 테스트 시나리오

- 스레드 수: 20개 (pool size의 2배)
- 상품: 재고 100개 (충분)
- 각 스레드: 서로 다른 유저로 1개씩 주문
- 기대: 일부 스레드가 connection timeout으로 실패

## 테스트 파일

- `concurrency/ConnectionPoolExhaustionTest.kt`
- 기존 동시성 테스트와 동일한 패턴 (CountDownLatch + ExecutorService)
- 테스트 목적과 결과를 주석으로 상세 서술

## 후속 작업

1. 테스트 작성 + 실행
2. 커밋 (test: 비관적 락 커넥션 풀 고갈 테스트 추가)
3. PR markdown 작성
4. 블로그(MD + HTML)에 해당 섹션 추가
