# mtd_server_desc
---
description: 메타 토이 드래곤즈 사가 게임 서버 설명(MTD WebAPI 프로젝트 구조, 기술 스택, 데이터 흐름을 정리)
---

# Part 1. 프로젝트 전체 분석 요약

> **프로젝트명**: MTD Server  
> **한 줄 요약**: 블록체인/NFT 모바일 게임 백엔드 API 서버

---

## 어떤 프로젝트인가

모바일 게임 "Meta Toy Dragonz Saga"의 서버 역할을 하는 API 서버다.  
게임 앱에서 버튼을 누를 때마다 이 서버로 요청이 날아오고, 서버는 DB에서 데이터를 읽고 쓴 뒤 결과를 JSON으로 돌려준다.  
NFT 거래, 가챠, 길드, PvP 아레나, 드래곤 육성 등 게임의 모든 기능을 담당한다.

---

## 기술 스택 한눈에 보기

| 역할 | 기술 | 한 줄 설명 |
|------|------|-----------|
| **언어** | PHP 8.x | 별도 프레임워크 없이 직접 구조 설계 |
| **웹 서버** | Nginx + PHP-FPM | 앱 요청을 받아 PHP 코드로 넘겨주는 입구 |
| **데이터베이스** | MySQL | 유저 정보, 아이템, 게임 기록 등 영구 저장 |
| **빠른 캐시** | Redis | 자주 쓰는 데이터를 메모리에 올려두고 빠르게 조회 |
| **랭킹** | Redis Sorted Set | 아레나 순위 실시간 집계 |
| **에러 알림** | Slack Webhook | 서버 에러 발생 시 슬랙으로 즉시 알림 |
| **외부 연동** | Marblex API | NFT/블록체인 거래 처리 |
| **소셜 로그인** | Google / Apple OAuth | 구글·애플 계정으로 로그인 |
| **정적 분석** | PHPStan | 코드 오류를 배포 전에 자동으로 잡아줌 |

---

## 폴더 구조

```
mtd_webapi/
├── public/       ← 앱이 직접 호출하는 API 파일들 (예: arena/enter-route.php)
│   ├── route.php         ← 모든 요청의 입구 (일반 게임 API)
│   ├── route_mbx.php     ← NFT/블록체인 전용 입구
│   ├── route_wv.php      ← 웹뷰 전용 입구
│   └── {기능명}/         ← auth, gacha, arena, guild, travel 등 기능별 폴더
│
├── src/          ← 기능별 비즈니스 로직 함수 (account.php, dragon.php 등)
├── class/        ← 재사용 가능한 핵심 클래스 (User, Dragon, GameDB 등)
├── common/       ← 전체 공통: 경로 상수, 환경 설정, 열거형 정의
├── config/       ← 개발/QA/운영 환경별 설정 파일
├── luascript/    ← Redis에서 실행되는 Lua 스크립트 (원자적 연산용)
├── private/      ← 관리자용 내부 스크립트, cron 작업
└── sql/          ← DB 마이그레이션 SQL 파일
```

---

## 요청 처리 흐름

앱에서 API를 호출하면 아래 순서로 처리된다.

```
[게임 앱]
    ↓  HTTP 요청
[Nginx 웹서버]
    ↓  PHP로 전달
[route.php]  ← URL을 보고 어떤 기능 파일을 실행할지 결정
    ↓
[public/기능명/액션-route.php]  ← 예: arena/enter-route.php
    ↓  파라미터 검증, 세션 확인
[src/ 비즈니스 로직]  ← 실제 게임 규칙 처리
    ↓
[class/ 도메인 클래스]  ← DB 읽기/쓰기, Redis 조회
    ↓
[Response::ok()]  ← JSON 조립 후 앱으로 응답 전송
```

---

## 주요 클래스 역할

| 클래스 | 역할 |
|--------|------|
| `GameDB` | DB 연결 관리. 읽기/쓰기 서버 자동 선택, 트랜잭션, 재시도 |
| `Response` | API 응답 조립. ok/error/push 데이터 포함 JSON 반환 |
| `User` | 유저 정보 로드/저장, 재화 변동 관리 |
| `Dragon` | 드래곤 데이터 CRUD |
| `SBRedisManager` | Redis 연결 관리. 다수의 읽기 서버 부하 분산 |
| `DesignData` | 기획 데이터(CSV) Redis 캐시 관리 |
| `SessionId` | 로그인 세션 발급/검증 |
| `SBLogger` | 재화 변동 이력 로깅 (응답 전송 후 일괄 DB 기록) |

---

## 환경 구분

| 환경 | 설명 |
|------|------|
| `DEV` / `DEV25` | 개발 서버 |
| `QA` | 테스트 서버 |
| `LIVE_MAIN` | 실제 서비스 운영 서버 |

- DEV/QA: 어떤 도메인에서나 API 호출 허용
- LIVE: 특정 도메인에서만 API 호출 허용 (보안)

---

## API 종류 구분

| 구분 | 진입점 | 대상 |
|------|--------|------|
| 게임 API | `route_main.php` | 게임 앱 본체 |
| NFT/블록체인 API | `route_mbx.php` | Marblex 연동 |
| 웹뷰 API | `route_wv.php` | 인게임 웹브라우저 화면 |

---

# Part 2. 기술적 하이라이트

---

## 1. ArenaRank: Float 비트 패킹으로 다중 정렬 기준을 단일 Score에 인코딩

**파일**: `class/arena/arenarank.class.php` — `setRank()` L123-158

Redis Sorted Set의 `score`는 float 1개뿐이다. 그런데 PvP 랭킹은 "포인트 → 승률 → 승수"의 세 단계 정렬이 필요하다. 이 셋을 하나의 float에 압축한다.

```php
$BITS16 = (1 << 16) - 1;                             // 65535
$wr = (int)($BITS16 * $win / $games);                // 승률을 16비트로 정규화
$decimal = (($wr << 16) + ($BITS16 & $win)) / (1 << 32); // 32비트 → 0~1 소수
$score = $pt + $decimal;
// 예: 2500점, 승률 75%, 승수 30
// → 2500.0000... + 승률16bit<<16 + 승수16bit  / 2^32
```

- 정수부: 포인트(기본 정렬)
- 소수점 상위 16비트: 승률(동점 1차 타이브레이커)
- 소수점 하위 16비트: 승수(동점 2차 타이브레이커)
- Redis에 저장할 때는 `zAdd` 1회, 조회할 때는 `zRevRank` 1회로 끝

DB 조인이나 추가 쿼리 없이 Redis 단일 자료구조로 3단계 정렬을 O(log n)에 처리한다.

---

## 2. GameDB: Writer/Reader 자동 라우팅 + Read-after-Write 일관성

**파일**: `class/gamedb.class.php` — `executeMeekro()` L429-539

단일 진입점에서 SQL 파싱 → 연결 선택 → RAW 보호 → 에러 재시도를 모두 처리한다.

```
SQL 앞 8글자 파싱(최소 비용) → SELECT류 여부 판별
    ↓
읽기 → reader MDB, 쓰기 → writer MDB
    ↓
트랜잭션 중 reader 잡히면 → 강제 writer 교정 (in-TX 가드)
    ↓
write 완료 → hrtime(true) 기준 300ms 동안 SELECT도 writer 라우팅
             (MySQL Replica 복제 지연 대응 Read-after-Write)
```

- `hrtime(true)` 마이크로초 정밀도 타이머 사용
- 트랜잭션 내 reader 잘못 잡히는 케이스 방어 코드 포함

---

## 3. GameDB: Deadlock 자동 재시도 + Exponential Backoff with Jitter

**파일**: `class/gamedb.class.php` — `executeMeekro()` L479-537

```php
// 재시도 가능: Deadlock(1213), Lock wait(1205), 연결 끊김(2006, 2013), Too many connections(1040)
// 즉시 실패: read-only write 시도(1290)

$jitter = random_int((int)($backoff_us * -0.2), (int)($backoff_us * 0.2));
usleep(max(50_000, $backoff_us + $jitter)); // 최소 50ms
$backoff_us = min(2_000_000, $backoff_us * 2); // 상한 2초
```

지터가 없으면 동시 요청들이 같은 간격으로 재시도해 thundering herd가 반복된다.
±20% 지터로 재시도 시점을 분산시켜 deadlock 해소 확률을 높인다.

---

## 4. DesignData: Probabilistic Early Expiration (Cache Stampede 방지)

**파일**: `class/designdata.class.php` — `checkEarlyExpire()` L240-260

게임 밸런스 CSV 캐시 만료 시 여러 PHP-FPM 워커가 동시에 갱신하러 달려드는 Cache Stampede를 수학적으로 방지한다.

```php
// PEU 알고리즘 (XFetch, 2015 논문 기반)
// 남은 수명이 짧을수록 조기 갱신 확률이 지수적으로 증가
return $now - LIFETIME * PEU_BETA * log(mt_rand() / mt_getrandmax()) > $exp;
```

- `PEU_BETA * log(rand)` : 지수분포 난수 → (-∞, 0] 구간
- 남은 수명 비율이 이 값의 절댓값보다 작으면 조기 갱신
- 동일 알고리즘이 아레나 랭킹 보드(`ArenaRank::isScoreboardTooOld()`)에도 적용됨

---

## 5. UnitStat: Dirty Flag 기반 40종 스탯 지연 계산 + 계층적 속성 분배

**파일**: `class/combat/unitstat.class.php`

40종 이상의 스탯을 `status_flag[]`(dirty bit) 배열로 관리한다. 변경된 스탯만 재계산(`calcStatus()`)하며, 변경되지 않은 스탯은 캐시 값을 재사용한다.

계층적 스탯 분배 구조:
```
ALL_ELEMENT_DMG (전 속성 공격)
    → FIRE_DMG 계산 시: cur_type + all_type 합산
    → WATER_DMG, WIND_DMG, ... 각각 합산

ALL_ELEMENT_DMG_PIERCE (전 속성 관통)
    → FIRE_DMG_PIERCE, WATER_DMG_PIERCE, ... 각각 합산

PHYS_DMG_RESIS (물리 저항)
    → ATK_DMG_RESIS, CRI_DMG_RESIS 계산에 추가 합산
```

- 스탯 타입별로 `VALUE / PERCENT / ADD_VALUE` 연산 방식이 다름
- `statValue()`로 스탯 min/max 범위 클램핑
- 속성(element), 저항(resis), 관통(pierce) 간의 상호작용이 단일 `calcStatus()` 메서드에서 처리

---

## 6. GameDB::commit(): 도메인 훅으로 Response Push 자동 조립

**파일**: `class/gamedb.class.php` — `commit()` L334-363

트랜잭션이 최상위에서 커밋될 때만 각 도메인 클래스의 `onCommit()`을 호출한다.

```php
if (0 === self::$tx_depth) {
    User::onCommit();    // 변경된 재화만 push에 추가
    Dragon::onCommit();
    Building::onCommit();
    // ...
}
```

- `User::onCommit()`은 `push_gemstone`, `push_gold` 등 변경 여부 플래그를 보고 변경된 항목만 `Response::addPush()`로 클라이언트에 전달
- route 파일이 push 데이터를 직접 조립하지 않아도 됨 (관심사 분리)
- 롤백 시 `stick_until_us` 해제 → push도 발생하지 않음

---

## 7. SBLogger: Deferred Write 패턴으로 응답 지연 없는 로깅

**파일**: `class/sblogger.class.php`

요청 처리 중 발생하는 모든 재화 변동(gemstone, gold, item, arenaPoint 등)을 정적 큐(`$log_data`, `$log_tables`, `$log_query`)에만 누적한다.

```php
// 요청 처리 중: 큐에만 push
SBLogger::gemstone($change, $after);
SBLogger::item($ino, $change, $after, $reason);

// Response::ok() 내부: 응답 직전 일괄 DB 저장
SBLogger::writeLog();
echo $ret;  // 클라이언트에 응답
```

- 로그 쓰기가 응답 지연을 일으키지 않음
- 트랜잭션 롤백 시 큐도 무의미해져 불필요한 로그가 남지 않음
- user_log, query_log 등 다중 테이블에 배치 INSERT

---

## 8. ArenaRank: PEU + Redis NX 분산 락으로 랭킹 보드 동시 갱신 방지

**파일**: `class/arena/arenarank.class.php` — `isScoreboardTooOld()`, `obtainScoreboardLock()`

랭킹 보드 갱신 시 두 가지 문제를 동시에 해결한다:

**① Cache Stampede 방지 (PEU)**
```php
// 유효기간 내에도 확률적 조기 갱신 → 동시 만료 방지
$early = BOARD_LIFE_MAX * log(mt_rand() / mt_getrandmax()) / BOARD_PEU_I_BETA;
return $lastUpdate + BOARD_LIFE_MAX + $early < $now;
```

**② 동시 갱신 방지 (Redis NX 락)**
```php
// NX: 이미 존재하면 SET 실패 → 락 획득 실패로 처리
return (bool)SBRedisManager::getWriter()->set($lockKey, 1, ['NX', 'EX' => $exp]);
```

두 조건을 모두 통과한 단 하나의 워커만 랭킹 보드를 실제로 갱신한다.

---

## 9. SBRedisManager: Reader Pool 라운드로빈 + 다단계 장애 폴백

**파일**: `class/sbredismanager.class.php`

```
getReader() → Pool 라운드로빈 선택
    ↓ (실패)
다음 Reader 순차 시도 (n-1회)
    ↓ (전부 실패)
Writer 폴백 → 서비스 중단 없는 degraded mode
```

레거시 단일 엔드포인트 설정과 신규 `writer`/`readers` 구조 모두 지원하는 하위 호환성 포함.

---

## 10. SessionId: Redis Hash 버켓팅 + Lua evalSha 원자적 갱신

**파일**: `class/sessionid.class.php`

유저 100만(`HASH_MOD`) 단위로 해시셋에 묶어 Redis 키 인덱싱 오버헤드를 절감한다.

```
유저 1,234,567번 → SESSION:1 해시셋의 "1234567" 필드
```

세션 읽기 시 Lua `evalSha`로 `hget + expire`를 원자적 1 round-trip으로 처리한다.
스크립트 미존재 시 자동 로드 후 재시도하는 자가 복구 패턴 포함.

---

## 11. GameDBTransaction: RAII 패턴으로 자동 롤백 보장

**파일**: `class/gamedb.class.php` L729-750

```php
class GameDBTransaction {
    function __construct() { GameDB::startTransaction(); }
    function __destruct()  { if (!$this->committed) GameDB::rollback(); }
}

// 사용
$tx = GameDB::getScopedTransaction();
// ... 비즈니스 로직 ...
$tx->commit(); // 명시적 commit 없으면 스코프 이탈 시 자동 rollback
```

예외 발생, 조기 `return`, 어떤 경로로 종료되어도 미커밋 트랜잭션은 반드시 롤백된다.

---

## 12. GameDB::ensureRequestState(): PHP-FPM 재사용 대비 요청 경계 초기화

**파일**: `class/gamedb.class.php` — `ensureRequestState()` L47-63

PHP-FPM 워커는 프로세스를 재사용하므로 static 변수가 이전 요청의 값을 유지한다.
`REQUEST_TIME_FLOAT`를 마커로 사용해, 새 요청 진입 시 `in_tx`, `tx_depth`, `stick_until_us` 등을 초기화한다.

이것이 없으면 이전 요청에서 트랜잭션이 비정상 종료됐을 때 다음 요청에서 모든 SELECT가 writer로 가는 버그가 생긴다.

---

## 요약 테이블

| # | 분류 | 핵심 기술 | 이점 |
|---|------|-----------|------|
| 1 | **비즈니스** | ArenaRank float 비트 패킹 | 3단계 정렬을 Redis 1 score로, 추가 쿼리 0 |
| 2 | **DB 인프라** | GameDB W/R 라우팅 + RAW | Replica 지연 버그 원천 차단 |
| 3 | **DB 인프라** | Backoff + Jitter | Deadlock 자동 복구, thundering herd 방지 |
| 4 | **공용 인프라** | DesignData PEU | Cache stampede 수학적 방지 |
| 5 | **비즈니스** | UnitStat Dirty Flag | 40+ 스탯 선택적 재계산, 계층적 분배 |
| 6 | **설계** | commit() 도메인 훅 | 트랜잭션-push 일관성 자동 보장 |
| 7 | **시스템** | SBLogger Deferred Write | 로깅이 응답 지연에 미치는 영향 0 |
| 8 | **비즈니스** | 랭킹보드 PEU + NX 락 | Stampede + 동시 갱신 이중 방어 |
| 9 | **Redis 인프라** | Reader Pool + 폴백 | 장애 시 무중단 degraded mode |
| 10 | **Redis 인프라** | Session 버켓팅 + evalSha | 메모리 절약 + 원자적 갱신 |
| 11 | **설계** | GameDBTransaction RAII | 예외/조기 리턴에도 자동 롤백 |
| 12 | **시스템** | ensureRequestState() | FPM 재사용 stale 상태 방지 |

---

# Part 3. 핵심 업무 및 성과

> 블록체인/NFT 모바일 게임 백엔드 API 서버  
> PHP 8.x · MySQL · Redis · Nginx/PHP-FPM · Marblex(NFT) 연동

---

## 1. DB 읽기/쓰기 서버 자동 분리 라우팅 구현

**[상황]** DB 서버를 쓰기 전용과 읽기 전용으로 나눠 운영했는데, 데이터를 저장한 직후 바로 읽으면 복제가 아직 안 된 오래된 값이 반환되는 문제가 잠재해 있었다.

**[행동]** 모든 DB 쿼리가 지나가는 단일 진입 함수를 만들었다. SQL 문장을 보고 읽기/쓰기를 자동으로 판단해 적절한 서버로 보내고, 쓰기 완료 후 300ms 동안은 읽기도 쓰기 서버로 보내 항상 최신 데이터를 보장했다.

**[결과]** "방금 저장했는데 왜 안 보여요?" 류의 데이터 불일치 오류가 사라졌고, 이후 DB 서버를 추가하거나 구조를 바꿔도 기능 파일을 수정할 필요가 없는 구조가 됐다.

---

## 2. DB 충돌 자동 복구 로직 구현 (재시도 간격 분산)

**[상황]** 동시 접속이 많을 때 여러 요청이 같은 DB 데이터를 동시에 수정하려다 충돌(Deadlock)이 나는 경우가 있었다. 바로 재시도하면 같은 요청들이 다시 동시에 충돌하는 문제가 반복됐다.

**[행동]** 재시도할수록 대기 시간이 2배씩 늘어나도록 하고, 거기에 무작위 오차를 ±20% 추가했다. 요청들이 서로 다른 타이밍에 재시도하게 되어 충돌 확률을 낮췄다. 재시도해도 의미 없는 에러는 즉시 포기하도록 에러 유형별로 분기했다.

**[결과]** DB 충돌 발생 시 별도 운영 개입 없이 대부분 자동으로 해결된다.

---

## 3. PvP 아레나 랭킹 시스템 설계 — 3단계 정렬을 DB 추가 조회 없이 처리

**[상황]** PvP 순위는 포인트가 같으면 승률, 승률도 같으면 승수로 가려야 했다. 실시간 랭킹을 담당하는 Redis는 숫자 1개로만 순위를 매길 수 있어 3단계 정렬이 불가능해 보였다.

**[행동]** 하나의 소수점 숫자에 세 가지 기준을 모두 담는 방법을 설계했다. 정수 부분에 포인트를, 소수점 앞자리에 승률을, 소수점 뒷자리에 승수를 인코딩했다. 저장과 순위 조회 모두 Redis 호출 1번으로 끝난다.

**[결과]** DB 추가 조회나 별도 정렬 로직 없이 3단계 순위 정렬이 가능해졌고, 랭킹 응답 속도가 향상됐다.

---

## 4. 캐시 동시 만료로 인한 서버 과부하 방지

**[상황]** 게임 밸런스 데이터를 Redis에 캐시해 사용하는데, 캐시가 만료되면 수백 개의 서버 프로세스가 동시에 데이터를 다시 불러오려 해 순간적으로 서버에 큰 부하가 걸릴 수 있었다.

**[행동]** 만료 시각이 가까워질수록 갱신 확률이 점점 높아지는 알고리즘(XFetch, 2015년 논문)을 적용했다. 하나의 프로세스가 만료 전에 미리 캐시를 갱신하고, 나머지는 갱신된 캐시를 그냥 사용한다. 아레나 랭킹 보드 갱신에도 같은 방식에 Redis 잠금을 추가해 중복 실행 자체를 막았다.

**[결과]** 캐시 만료 시 순간 과부하 현상이 사라졌으며, 두 기능에 동일한 패턴을 재사용해 일관된 설계를 유지했다.

---

## 5. 트랜잭션 완료 시 클라이언트 응답 데이터 자동 조립 구조 설계

**[상황]** 보석·골드 등 재화 변동이 생기면 앱에 최신 값을 push로 알려줘야 했다. 각 기능 파일에서 개별적으로 push 데이터를 만들다 보니 누락되거나 중복 전송되는 경우가 생겼다.

**[행동]** DB 트랜잭션이 최종 완료될 때 각 도메인 클래스(유저, 드래곤 등)가 "나 변경됐어요"를 자동으로 응답에 추가하는 구조를 만들었다. 트랜잭션이 취소될 경우 push도 자동으로 취소된다.

**[결과]** 기능 개발 시 push 데이터를 별도로 신경 쓰지 않아도 되어 개발 실수가 줄었고, 데이터 저장과 앱 응답의 일관성이 구조적으로 보장됐다.

---

## 6. 로그 기록이 API 응답 속도에 영향을 주지 않는 구조 구현

**[상황]** 재화 변동 이력을 요청 처리 중에 바로 DB에 기록하면 로그 쓰기 시간만큼 응답이 느려졌다.

**[행동]** 요청 처리 중에는 로그를 메모리에만 쌓아두고, 앱에 응답을 보낸 직후 일괄로 DB에 기록하도록 구조를 바꿨다.

**[결과]** 로그 저장이 응답 속도에 미치는 영향이 없어졌다. DB 오류로 처리가 취소됐을 때도 잘못된 로그가 남지 않는 효과도 얻었다.

---

## 7. PHP 서버 재사용 환경에서 요청 간 상태 오염 방지

**[상황]** PHP 서버는 프로세스를 재사용하기 때문에, 이전 요청에서 비정상 종료가 생기면 그 상태가 다음 요청까지 남아 엉뚱한 DB 서버로 쿼리가 가는 버그가 발생할 수 있었다.

**[행동]** 각 요청이 시작될 때 이전 요청과 같은 프로세스인지 확인하고, 새 요청이면 관련 상태를 전부 초기화하는 코드를 DB 진입 함수에 추가했다.

**[결과]** 장시간 운영해도 이전 요청의 상태가 영향을 주는 버그가 발생하지 않았다.

---

## 8. 트랜잭션 미완료 자동 취소 구조 도입

**[상황]** 코드에서 에러가 나거나 중간에 함수를 일찍 종료하면 DB 트랜잭션이 완료되지 않은 채 남아 데이터가 잠기는 문제가 생길 수 있었다. 개발자가 매번 "취소(롤백)" 코드를 작성해야 했고 빠뜨리는 경우도 있었다.

**[행동]** 트랜잭션을 시작하면 객체가 생성되고, 그 객체가 사라질 때 자동으로 취소하는 구조를 만들었다. 명시적으로 "완료(커밋)"을 호출했을 때만 정상 완료된다.

**[결과]** 어떤 방식으로 코드가 종료되어도 미완료 트랜잭션은 항상 자동 취소되어 데이터 안전성이 보장됐다. 개발자가 취소 코드를 일일이 신경 쓰지 않아도 되는 안전한 패턴이 팀 전체에 적용됐다.

---

## 9. Redis 서버 다중화 및 장애 시 자동 전환

**[행동]** Redis 읽기 서버를 여러 대 운영하면서 요청이 고르게 분산되도록 했다. 특정 서버가 응답하지 않으면 다른 서버로 자동으로 넘어가고, 읽기 서버 전체가 다 죽어도 쓰기 서버로 전환해 서비스를 이어갈 수 있도록 했다.

**[결과]** Redis 일부 장애 시 서비스 중단 없이 운영이 유지됐다. 기존 단일 서버 설정을 그대로 유지하면서 신규 다중 서버 구조로 전환할 수 있어 배포 리스크도 없었다.

---

## 10. 전투 스탯 시스템 — 변경된 것만 재계산하는 구조

**[행동]** 40종 이상의 전투 스탯을 관리하면서, 값이 바뀐 스탯에만 "변경 표시"를 해두고 표시된 것만 재계산하도록 했다. "전체 속성 공격력"을 올리면 "불 공격력", "물 공격력" 등 하위 스탯에 자동으로 반영되는 계층 구조도 설계했다.

**[결과]** 전투 중 불필요한 계산이 줄어들었고, 복잡한 속성 간 상호작용이 한 곳에서 일관되게 처리되어 유지보수가 쉬워졌다.

---

## 기술 스택 요약

| 분류 | 기술 |
|------|------|
| **언어** | PHP 8.x (프레임워크 없이 직접 설계) |
| **웹 서버** | Nginx + PHP-FPM |
| **데이터베이스** | MySQL, 읽기/쓰기 서버 분리, 3개 게임 서버 샤딩 |
| **캐시/랭킹** | Redis (정렬 집합, 해시, Lua 스크립트) |
| **설계 패턴** | 단일 진입점 라우팅, 자동 롤백, 지연 쓰기, 도메인 훅 |
| **안정성 기법** | 재시도 간격 분산, 확률적 조기 갱신, Redis 잠금, 변경 감지 |
| **외부 연동** | Marblex API(NFT/블록체인), Google/Apple 소셜 로그인, IMX |
| **정적 분석** | PHPStan |
| **모니터링** | Slack 알림 (에러 발생 시 자동 전송) |

