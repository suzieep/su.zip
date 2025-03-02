# @Transactional 적용 범위와 설정

## @Transactional 사용시 주의점

* Spring의 Transaction 관리 방식에서 Proxy 기반의 AOP을 사용
* **같은 Class 내에서 호출된 method는 Proxy를 거치지 않음** -> Transaction 적용 X
* **private Method는 외부에서 접근해서 Proxy 객체 생성이 불가능** -> Transaction 적용 X ㄴ 하지만 @Transactional 붙어있는 public method에서 private method 호출하면 적용 O

## @Transactional 동작 예시

#### Case 1) outerMethod(@tr)가 같은 클래스에 있는 innerMethod를 호출할 때

```java
public class UserService {
    @Transactional
    public void mainMethod() {
        createUser();
        updateUser();
        deleteUser();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    private void createUser() {}

    @Transactional
    private void updateUser() {}

    private void deleteUser() {}
}
```

* 다른 클래스에서 `mainMethod()`가 호출되면 `tr1`이 시작된다
* createUser/updateUser 가 호출되어도 같은 class에서 호출하기 때문에 @Transactional이 걸리지 않아 그대로 `tr1`에 참여한다
* deleteUser 또한 상위 메소드 Transaction이 열려있기 때문에 `tr1`을 사용
* mainMethod가 종료될 때 Commit

### Case 2) outerMethod(@tr)에서 다른 클래스에 있는 innerMethod를 호출할 때

```java
public class UserService {
    @Transactional
    public void mainMethod() {
        subService.createUser();
        subService.updateUser();
        subService.deleteUser();

    }
}

public class SubService  {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    private void createUser() {}

    @Transactional(propagation = Propagation.REQUIRED)
    private void updateUser() {}

    private void deleteUser() {}
}
```

* 다른 클래스에서 `mainMethod()`가 호출되면 `tr1`이 시작된다
* `createUser()` : `tr1`이 중지되고 `tr2` 생성 -> 마치면 commit + tr1 재개
* `updateUser()` : `tr1`에 참여
* `deleteUser()` : `tr1`에 참여

### Case 3) outerMethod(@tr X)에서 다른 클래스에 있는 innerMethod를 호출할 때

```java
public class UserService {
    public void mainMethod() {
        subService.createUser();
        subService.updateUser();
        subService.deleteUser();

    }
}

public class SubService  {
    @Transactional(propagation = Propagation.REQUIRED)
    private void createUser() {}

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    private void updateUser() {}

    private void deleteUser() {}
}
```

* 다른 클래스에서 `mainMethod()`가 호출
* `createUser()` : `tr1` 생성 -> 함수 마칠 때 종료(commit)
* `updateUser()` : `tr1` 생성 -> 함수 마칠 때 종료(commit)
* `deleteUser()` : transaction 없이 실행(DB작업 즉시 커밋)

## Transaction Attributes

```java
@Transactional(
    isolation = Isolation.REPEATABLE_READ, 
    propagation = Propagation.REQUIRED, 
    timeout = 30, 
    readOnly = true
)
```

### Isolation

1. DEFAULT - 데이터베이스의 기본 격리 수준(일반적으로 READ\_COMMITTED)
2. READ\_UNCOMMITTED - 커밋되지 않은 데이터를 읽을 수 있음 ㄴ Dirty Read
3. **READ\_COMMITTED** - 커밋된 데이터만 읽을 수 있음 ㄴ Non-repeatable Read - 동일한 트랜잭션 내에서 같은 데이터를 다시 읽을 때 값이 달라질 수 있음.
4. REPEATABLE\_READ - 트랜잭션 내에서 같은 데이터를 반복해서 읽을 때 동일한 결과를 보장 ㄴ Phantom Read - 트랜잭션 중에 다른 트랜잭션이 데이터를 삽입해 조회 시 추가 데이터가 나타날 수 있음.
5. SERIALIZABLE - 트랜잭션 직렬화(순차 실행)

### Propagation

1. **REQUIRED**(Default) - Transaction이 존재하면 참여하고, 없으면 생성
2. **REQUIRES\_NEW** - 새 Transaction 시작, 이미 존재하는 Transaction 중지
3. SUPPORTS - 있으면 참여, 없으면 없는대로
4. NOT\_SUPPORTED - 있으면 중지, 항상 없이 실행
5. MANDATORY - 현재 진행중인 tr 참여, 없으면 IllegalTransactionStateException
6. NEVER - 현재 진행중인 tr 있으면 IllegalTransactionStateException, 항상 없이 실행
7. **NESTED** - 현재 진행중인 tr에 중첩된 tr 생성(기존 tr의 일부로 취급) ㄴ savepoint 사용해 rollback 가능

### Timeout

* Transaction 최대 실행 시간 설정
* default : -1, 기본적으로 시간 제한 없음

### ReadOnly

* 트랜잭션을 읽기 전용으로 설정해서, 데이터 수정 작업 수행하지 않도록 막음
* 성능 최적화와 충돌 방지
* default : false

### References

* https://hungseong.tistory.com/81
* https://green-bin.tistory.com/79
