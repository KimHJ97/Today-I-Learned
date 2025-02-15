# Redis 기반 선착순 쿠폰 발급 서버 설계

## 쿠폰 발급 서버 구조 학습

 - `쿠폰 API 서버 흐름`
```
User -> API Server -> Database
1. 유저가 요청을 보낸다.
2. API 서버에서는 요청을 처리한다.
3. 요청을 처리하는 과정에서 데이터베이스를 사용하여 트랜잭션을 처리한다.

쿠폰 발급 트랜잭션
 - 1. 쿠폰 조회
 - 2. 쿠폰 발급 내역 조회
 - 3. 쿠폰 수량 증가 & 쿠폰 발급
```
<br/>

 - `쿠폰 API 서버 구조 개선`
```
User -> API Server -> 쿠폰 발급 요청 -> Redis -> 쿠폰 발급 대기열 적제 -> Queue
1. N명의 유저가 요청을 보낸다.
2. API 서버에서는 요청을 처리한다.
3. Redis에서 요청을 처리하고 쿠폰 발급 대상을 저장한다.
4. 쿠폰 발급 처리 기능에서 Redis의 쿠폰 발급 대상을 조회하여 발급 처리한다.
 - 쿠폰 발급 서버를 분리할 수 있다.

Redis(Pulling) -> 발급 Server -> 쿠폰 발급 -> MySQL
```
<br/>

### Key Point와 Action Plan

 - __Key Point__
    - 유저 트래픽과 쿠폰 발급 트랜잭션 분리
        - Redis를 통한 트래픽 대응
        - MySQL 트래픽 제어
    - 비동기 쿠폰 발급 시스템
        - Queue를 인터페이스로 발급 요청/발급 과정을 분리
    - 쿠폰 발급 요청을 처리하는 API 서버/쿠폰 발급을 수행하는 발급 서버 분리
 - __Action Plan__
    - MySQL 조회 없이 Redis만 사용해서 사용자 요청을 처리할지?
        - 쿠폰 기한 검증
        - 쿠폰 발급 수량 검증
        - 중복 발급 검증
    - Redis로 어떻게 처리할 수 있을까?

<br/>

## 적절한 Redis 데이터 구조 결정하기

Redis를 통해서 유저의 요청에 대한 트래픽을 처리해야 한다.  

<br/>

### 쿠폰 발급 기능 분석

 - __검증__
    - 쿠폰 존재 검증
    - 쿠폰 발급 유효 기간 검증
    - 쿠폰 발급 수량 검증
    - 중복 발급 검증
 - __발급__
    - 쿠폰 발급 수량 증가
    - 쿠폰 발급 내역 기록

<br/>

### Redis Cache 고민해보기

 - __Coupon 엔티티 캐시__
    - 검증에 활용할 수 있는가?
        - 쿠폰 존재 검증 [O]
        - 쿠폰 발급 유효 기간 검증 [O]
        - 쿠폰 발급 수량 검증 [X]
            - 발급 수량은 실시간성으로 변경이 되는 값이다.
    - Coupon 엔티티 캐시 활용
        - 쿠폰 존재 검증
        - 쿠폰 발급 유효 기간 검증

<br/>

 - __쿠폰의 발급 수량 관리__
    - 쿠폰 발급 수량 검증
    - 중복 발급 검증
    - 쿠폰 발급 수량 증가
    - 쿠폰 발급 내역 기록

<br/>

#### LIST

 - Stack, Queue 구현 가능
 - 중복을 허용
 - __선착순 조건 처리__
    - 유저 요청에 대한 Queue 구현
        - 1. 유저 요청(coupon_id, user_id)
        - 2. List에 요청을 추가 (RPUSH)
            - 요청에 따라 Queue를 구현할 수 있음
 - __발급 수량 관리__
    - 1. 유저 요청이 들어옴 (coupon_id, user_id)
    - 2. 발급 수량 조회 및 검증 (LLEN)
        - List는 중복을 허용하므로 Unique한 요청의 수를 구분하기 어렵다.
    - 3. 중복 발급 검증 (LPOS)
        - O(N) 탐색이 지속적으로 필요

선착순 요청에 따라 Queue를 구현하는 과정이 매우 간단하여 효율적이다.  
다만, 발급 수량 검증 과정이 매우 까다롭다.  

<br/>

#### SET

 - 중복을 허용하지 않음
 - 순서를 보장하지 않음
 - __선착순 조건 처리__
    - 1. 유저 요청이 들어옴 (coupon_id, user_id)
    - 2. Set에 요청을 추가 (SADD)
        - 순서를 알 수 없다. 떄문에, 선착순에 대한 처리를 할 수 없다.
 - __발급 수량 관리__
    - 1. 유저 요청이 들어옴 (coupon_id, user_id)
    - 2. 발급 수량 조회 및 검증 (SCARD)
        - Set은 중복을 허용하지 않으므로 Unique한 요청의 수를 구분하기 쉽다.
    - 3. 중복 발급 검증 (SISMEMBER, SADD)
        - O(1)로 탐색이 가능하다.

요청의 순서를 구분할 수 없다.  
발급 수량 검증은 간단하다.  

<br/>

#### SORTED SET

 - 중복을 허용하지 않음
 - 스코어를 사용하여 정렬 가능
 - __선착순 조건 처리__
    - 1. 유저 요청이 들어옴 (coupon_id, user_id)
    - 2. Sorted Set에 요청을 추가 (ZADD)
        - score를 time stamp로 활용하면 요청 순서로 정렬할 수 있음
 - __발급 수량 관리__
    - 1. 유저 요청이 들어옴 (coupon_id, user_id)
    - 2. 발급 수량 조회 및 검증 (ZCARD)
        - Sorted Set은 중복을 허용하지 않으므로 Unique한 요청의 수를 구분하기 쉽다.
    - 3. 중복 발급 검증 (ZRANK, ZADD)

score를 활용해서 요청의 순서를 구분할 수 있다. 다만, O(longN) 복잡도가 소요된다.  
발급 수량 검증이 간단하다.  

<br/>

### Sorted Set을 활용한 구조

 - 예상 시나리오
```
1. 유저 요청이 들어온다. (coupon_id, user_id)
2. 쿠폰 캐시를 통한 유효성 검증
 - 쿠폰의 존재
 - 쿠폰의 유효기간

3. Sorted Set에 요청을 추가 (ZADD score = time stamp)
 - ZADD의 응답 값 기반 중복 발급 검증
4. 현재 요청의 순서 조회 (ZRANK) 및 발급 성공 여부 응답
5. 발급에 성공했다면 쿠폰 발급 Queue에 적재
```
<br/>

### SET을 활용한 구조

 - 예상 시나리오
```
1. 유저 요청이 들어온다. (coupon_id, user_id)
2. 쿠폰 캐시를 통한 유효성 검증
 - 쿠폰의 존재
 - 쿠폰의 유효기간

3. 중복 발급 요청 여부 확인 (SISMEMBER)
4. 수량 조회 (SCARD) 및 발급 가능 여부 검증
5. 요청 추가 (SADD)
6. 쿠폰 발급 Queue에 적재
```
