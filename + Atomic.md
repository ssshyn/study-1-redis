Atomic = 원자적 : 중간 상태 없이 완전히 실행되거나 전혀 실행되지 않음


##### 문제 상황 — Java 코드로 증가시키면 어떻게 될까?

```java
// 애플리케이션 레벨에서 증가 시도 (위험!)
String raw = redisTemplate.opsForValue().get("counter");
long value = Long.parseLong(raw);  // ① 읽기
value++;                            // ② 계산
redisTemplate.opsForValue().set("counter", value); // ③ 쓰기
```

서버 인스턴스가 2개(A, B) 동시에 이 코드를 실행하면:

```
초기값: 10

A: ① 읽기 → 10
B: ① 읽기 → 10   ← A가 쓰기 전에 B도 읽어버림
A: ② 계산 → 11
B: ② 계산 → 11
A: ③ 쓰기 → 11
B: ③ 쓰기 → 11   ← 11로 덮어씀 (12가 되어야 하는데!)

결과: 10 → 11  (카운트가 1만 증가, Lost Update 발생)
```

이게 **Race Condition**이다. 두 번 증가했는데 한 번만 반영됐다.

##### Redis INCR이 Atomic한 이유

Redis는 **싱글 스레드로 커맨드를 순차 처리**한다.  
`INCR`은 "읽기 + 계산 + 쓰기"가 **하나의 커맨드**로 처리되므로  
두 클라이언트가 동시에 보내도 커맨드 큐에서 차례를 기다린다.

```
초기값: 10

A: INCR counter → [읽기:10, +1, 쓰기:11] 원자적 실행 → 11 반환
B: INCR counter → [읽기:11, +1, 쓰기:12] 원자적 실행 → 12 반환

결과: 10 → 12  (정확하게 2번 증가)
```

##### Atomic이 보장되는 Redis 커맨드들

| 커맨드 | 동작 | Atomic 이유 |
|---|---|---|
| `INCR` / `INCRBY` | 읽기 + 증가 + 쓰기 | 단일 커맨드 |
| `SETNX` (= `SET NX`) | 존재 확인 + 저장 | 단일 커맨드 |
| `GETSET` | 읽기 + 저장 | 단일 커맨드 |
| `LPUSH` + `LTRIM` | 별개 커맨드 2개 | **Atomic 아님** → Pipeline/MULTI 필요 |

##### MULTI/EXEC — 여러 커맨드를 묶어서 Atomic하게

단일 커맨드가 아닌 여러 커맨드를 묶으려면 트랜잭션을 써야 한다.

```bash
MULTI           # 트랜잭션 시작
SET key1 "a"
INCR counter
SET key2 "b"
EXEC            # 큐에 쌓인 커맨드 한 번에 실행
```

RedisTemplate에서는:

```java
List<Object> results = redisTemplate.execute(new SessionCallback<>() {
    @Override
    public List<Object> execute(RedisOperations operations) {
        operations.multi();                                    // MULTI
        operations.opsForValue().set("key1", "a");
        operations.opsForValue().increment("counter");
        operations.opsForValue().set("key2", "b");
        return operations.exec();                             // EXEC
    }
});
```

> **주의**: Redis 트랜잭션은 RDB 트랜잭션과 다르다.  
> 커맨드 실행 중 오류가 나도 **롤백이 없다**. 이미 실행된 커맨드는 그대로 반영된다.  
> "Atomic"의 의미가 "다른 클라이언트의 커맨드가 중간에 끼어들지 못한다"는 격리성에 가깝다.