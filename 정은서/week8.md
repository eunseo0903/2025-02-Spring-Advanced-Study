# [정은서] 2025 GDG Spring Advanced Study - 8주차
# 토비의 스프링 3.1(vol1) - 6장 AOP

## AOP(3)

### 6.6 트랜잭션 속성

트랜잭션을 가져올 때 파라미터로 트랜잭션 매니저에게 전달하는 DefaultTransactionDefinition의 용도가 무엇인지 알아보자

6.6.1 트랜잭션 정의

- 트랜잭션은 더 이상 쪼갤 수 없는 최소 단위의 작업
- 트랜잭션 경계 안에서 진행된 작업은 commit()을 통해 모두 성공하든지 아니면 rollback()을 통해 모두 취소돼야 함
- 트랜잭션의 동작방식을 제어할 수 있는 조건
1. 트랜잭션 전파: 트랜잭션의 경계에서 이미 진행 중인 트랜잭션에 있을 때 또는 없을 때 어떻게 동작할 것인가를 결정하는 방식
    - PROPAGATION_REQUIRED: 진행 중인 트랜잭션이 없으면 새로 시작하고, 이미 시작된 트랜잭션이 있으면 이에 참여
    - PROPAGATION_REQUIRED_NEW: 항상 새로운 트랜잭션을 시작한다.
    - PROPAGATION_NOT_SUPPORTED: 진행 중인 트랜잭션이 있어도 무시한다. 특별한 메소드만 트랜잭션 적용에서 제외하려는 경우
2. 격리수준
    - 가능하다면 모든 트랜잭션이 순차적으로 진행돼서 다른 트랜잭션의 작업에 독립적인 것이 좋겠지만 그러면 성능이 크게 떨어질 수 있음
    - 적절하게 격리수준을 조정해서 가능한 한 많은 트랜잭션을 동시에 진행시키면서도 문제가 발생하지 않게 하는 제어가 필요
3. 제한시간
    - 트랜잭션을 수행하는 제한시간을 설정할 수 있다
4. 읽기전용
    - 트랜잭션 내에서 데이터를 조작하는 시도를 막아줄 수 있다

원하는 메소드만 선택해서 독자적인 트랜잭션 정의를 적용할 수 있는 방법은 없을까?

6.6.2 트랜잭션 인터셉터와 트랜잭션 속성

- TransactionInterceptor
    - 트랜잭션 정의를 메소드 이름 패턴을 이용해서 다르게 지정할 수 있는 방법을 추가로 제공
    - PlatformTransactionManager와 Properties 타입의 두 가지 프로퍼티를 갖고 있다

```java
public Object invoke(MethodInvocation invocation) throws Throwable {
        // (트랜잭션 전파, 격리 수준, 타임아웃, 읽기 전용 여부 등)
        TransactionStatus status = 
            this.transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            Object ret = invocation.proceed(); 
            this.transactionManager.commit(status);
            return ret;
        } catch (RuntimeException e) { // 롤백 대상인 예외 종류
            this.transactionManager.rollback(status);
            throw e;
        }
        // 이 두 가지가 결합해서 트랜잭션 부가기능의 행동을 결정하는 TransactionAttribute 속성이 됨 
}
```

- TransactionInterceptor의 두 가지 예외 처리 방식
    - 런타임 예외가 발생가면 트랜잭션은 롤백된다
    - 타깃 메소드가 런타임 예외가 아닌 체크 예외를 던지는 경우에는 이것을 예외상황으로 해석하지 않고 일종의 비즈니스 로직에 따른, 의미가 있는 리턴 방식의 한 가지로 인식해서 트랜잭션을 커밋
- TransactionAttribute의 rollbackOn() 속성
    - 특정 체크 예외의 경우는 트랜잭션을 롤백
    - 특정 런타임 예외에 대해서는 트랜잭션을 커밋
- 메소드 이름 패턴을 이용한 트랜잭션 속성 지정

```java
<bean id="transactionAdvice" 
    class="org.springframework.transaction.interceptor.TransactionInterceptor">
    
    <property name="transactionManager" ref="transactionManager" />
    
    <property name="transactionAttributes">
        <props>
            <prop key="get*">PROPAGATION_REQUIRED,readOnly,timeout_30</prop>
            <prop key="upgrade*">PROPAGATION_REQUIRES_NEW,ISOLATION_SERIALIZABLE</prop>
            <prop key="*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
</bean>
```

때로는 메소드 이름이 하나 이상의 패턴과 일치하는 경우가 있다. 이때는 메소드 이름 패턴 중에서 가장 정확히 일치하는 것이 적용된다. 이렇게 메소드 이름 패턴을 사용하는 트랜잭션 속성을 활용하면 하나의 트랜잭션 어드바이스를 정의하는 것만으로도 다양한 트랜잭션 설정이 가능해진다. 

6.6.3 포인트컷과 트랜잭션 속성의 적용 전략

- 트랜잭션 부가기능을 적용할 후보 메소드를 선정하는 작업은 포인트컷에 의해 진행됨
- 어드바이스의 트랜잭션 전파 속성에 따라서 메소드 별로 트랜잭션의 적용 방식이 결정
- 포인트컷 표현식과 트랜잭션 속성을 정의할 때 따르면 좋은 몇 가지 전략
    - 트랜잭션 포인트컷 표현식은 타입 패턴이나 빈 이름을 이용
    - 공통된 메소드 이름 규칙을 통해 최소한의 트랜잭션 어드바이스와 속성을 정의
        - 모든 메소드에 대해 디폴트 속성을 부여
        
        ```java
        <tx:advice id="transactionAdvice">
            <tx:attributes>
                <tx:method name="*" /> // 모든 타깃 메소드에 기본 트랜잭션 속성 지정 
            </tx:attributes>
        </tx:advice>
        ```
        
    - 프록시 방식 AOP는 같은 타깃 오브젝트 내의 메소드를 호출할 때는 적용되지 않는다.
        - 프록시 방식의 AOP에서는 프록시를 통한 부가 기능의 적용은 클라이언트로부터 호출이 일어날 때만 가능하다
        - 같은 오브젝트 안에서의 호출은 새로운 트랜잭션 속성을 부여하지 못한다는 사실을 의식해야 함

6.6.4 트랜잭션 속성 적용

트랜잭션 속성과 그에 따른 트랜잭션 전략을 UserService에 적용해보자 

```java
public interface UserService {
    
    void add(User user);
    
    // 신규 추가 메서드 (CRUD 기본 기능)
    User get(String id);
    List<User> getAll();
    void deleteAll();
    void update(User user);

    void upgradeLevels();
}
```

```java
public class UserServiceImpl implements UserService {
    
    // DAO에 작업을 위임하기 위한 객체 (의존성 주입 대상)
    UserDao userDao; 
    
    public void deleteAll() { 
        userDao.deleteAll(); 
    }
    
    public User get(String id) { 
        return userDao.get(id); 
    }
    
    public List<User> getAll() { 
        return userDao.getAll(); 
    }
    
    public void update(User user) { 
        userDao.update(user); 
    }
}
```

upgradeLevels()에만 트랜잭션이 적용되게 했던 기존 포인트컷 표현식을 모든 비즈니스 로직의 서비스 빈에 적용되도록 수정

다음은 TransactionAdvice 클래스로 정의했던 어드바이스 빈을 스프링의 TransactionInterceptor를 이용하도록 변경

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx-3.0.xsd">

    <tx:advice id="transactionAdvice">
        <tx:attributes>
            <tx:method name="get*" read-only="true" />
            <tx:method name="*" />
        </tx:attributes>
    </tx:advice>
</beans>
```

### 6.7 애노테이션 트랜잭션 속성과 포인트컷

가끔은 클래스나 메소드에 따라 제각각 속성이 다른 세밀하게 튜닝된 트랜잭션 속성을 적용해야 하는 경우도 있다. → 직접 타깃에 트랜잭션 속성 정보를 가진 애노테이션을 지정하는 방법

6.7.1 트랜잭션 애노테이션

 @Transactional

타깃은 메소드와 타입이다. 애노테이션을 트랜잭션 속성 정보로 사용하도록 지정하면 스프링은  @Transactional이 부여된 모든 오브젝트를 자동으로 타깃 오브젝트로 인식한다. 

```java
package org.springframework.transaction.annotation;

// ... (애너테이션을 사용할 대상을 지정한다. 여기에 사용된 '메서드와 타입(클래스, 인터페이스)'처럼 한 개 이상의 대상을 지정할 수 있다.)
@Target({ElementType.METHOD, ElementType.TYPE})

// 애너테이션 정보가 언제까지 유지되는지를 지정한다. 이렇게 설정하면 런타임 때도 애너테이션 정보를 리플렉션을 통해 얻을 수 있다.
@Retention(RetentionPolicy.RUNTIME)

// 상속을 통해서도 애너테이션 정보를 얻을 수 있게 한다.
@Inherited

@Documented
```

```java
public @interface Transactional {
    
    String value() default "";

    Propagation propagation() default Propagation.REQUIRED;
    Isolation isolation() default Isolation.DEFAULT;
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
    boolean readOnly() default false;

    Class<? extends Throwable>[] rollbackFor() default {};
    String[] rollbackForClassName() default {};
    Class<? extends Throwable>[] noRollbackFor() default {};
    String[] noRollbackForClassName() default {};
}
// 트랜잭션 속성의 모든 항목을 엘리먼트로 지정할 수 있다.
// 디폴트 값이 설정되어 있으므로 모두 생략 가능하다.
```

- 트랜잭션 속성을 이용하는 포인트컷

메소드마다 @Transactional을 부여하고 속성을 지정할 수 있다. 이렇게 하면 유연한 속성 제어는 가능하겠지만 코드는 지저분해지고, 동일한 속성 정보를 가진 애노테이션을 반복적으로 메소드마다 부여해주는 바람직하지 못한 결과를 가져올 수 있음

- 대체 정책
    - 스프링은 @Transactional을 이용할 때 4단계의 대체 정책을 이용하게 해줌
    - 메소드의 속성을 확인할 때 타깃메소드, 타깃 클래스, 선언 메소드, 선언 타입의 순서에 따라서 @Transactional이 적용됐는지 차례로 확인하고 가장 먼저 발견되는 속성정보를 사용
    - 애노테이션 자체는 최소한으로 사용하면서도 세밀한 제어가 가능

6.7.2 트랜잭션 애노테이션 적용 

```java
@Transactional
public interface UserService {
    
    // <tx:method name="*" />와 같은 설정 효과를 가져온다.
    void add(User user);
    void deleteAll();
    void update(User user);
    void upgradeLevels();
    
    // @Transactional 애너테이션이 없으므로 대체 정책에 따라 타입 레벨에 부여된 디폴트 속성이 적용된다.
    // (이 부분은 인터페이스 레벨에 @Transactional이 있으므로, 모든 메서드에 해당 속성이 적용됨을 의미합니다.)
    
    // <tx:method name="get*" read-only="true"/>를 애너테이션 방식으로 변경한 것이다.
    @Transactional(readOnly=true)
    User get(String id);
    
    @Transactional(readOnly=true)
    List<User> getAll();
    
    // 메서드 단위로 부여된 트랜잭션의 속성이 타입 레벨에 부여된 것에 우선해서 적용된다. 
    // 같은 속성을 가졌더라도 메서드 레벨에 부여될 때는 메서드마다 반복될 수밖에 없다.
}
```

### 6.8 트랜잭션 지원 테스트

6.8.1 선언적 트랜잭션과 트랜잭션 전파 속성

트랜잭션을 정의할 때 지정할 수 있는 트랜잭션 전파 속성은 매우 유용한 개념이다. 예를 들어 required로 전파 속성을 지정해줄 경우, 앞에서 진행 중인 트랜잭션이 있으면 참여하고 없으면 자동으로 새로운 트랜잭션을 시작해준다.  

- REQUIRED 전파 속성을 가진 메소드를 결합해 다양한 크기의 트랜잭션 작업을 만들 수 있음
- A 메소드에서 B 메소드를 호출하는데, A와 B 작업이 모두 완료되어야만할 때, REQUIRED 전파 속성을 사용해야 함
- AOP를 이용해 코드 외부에서 트랜잭션 기능을 부여해주는 방법을 선언적 트랜잭션(Declarative Transaction)이라 함
- 대조적으로, TransactionTemplate나 개별 데이터 트랜잭션 API를 사용해 직접 코드 안에서 사용하는 방법을 프로그램에 의한 트랜잭션(Programmatic Transaction)이라 함
- 특수한 상황을 제외하고, 선언적 트랜잭션 방법을 사용하는 것을 추천함

6.8.2 트랜잭션 매니저와 트랜잭션 동기화

- AOP 덕분에 트랜잭션 부가기능을 간단히 애플리케이션 전반에 적용할 수 있음
- AOP의 중요한 기술적 기반은 트랜잭션 추상화임
- 트랜잭션 추상화의 핵심은 트랜잭션 매니저와 트랜잭션 동기화임
- PlatformTransactionManager 인터페이스를 구현한 트랜잭션 매니저를 통해 구체적인 트랜잭션 기술의 종류에 상관없이 일관된 트랜잭션 제어가 가능
- TransactionSynchronization을 사용하는 트랜잭션 동기화는 같은 DB Connection을 공유하게 함
- 즉, 트랜잭션 동기화 기술은 DB Connection을 공유하는 특성을 이용하여 트랜잭션 전파 속성에 따라 참여할 수 있도록 함

6.8.3 테스트를 위한 트랜잭션 애노테이션

@Transactional 애노테이션을 타깃 클래스 또는 인터페이스에 부여하는 것만으로 트랜잭션을 적용해주는 것은 매우 편리한 기술이고 이 압법을 테스트 클래스와 메스드에도 적용할 수 있다. 

```java
@Test
@Transactional
    public void transactionSync(){
        userService.deleteAll();
        userService.add(users.get(0));
        userService.add(users.get(1));
    }
```

테스트에 적용된 @Transactional은 기본적으로 트랜잭션을 강제 롤백시키도록 설정되어 있음 (롤백 테스트)

@Transactional은 테스트 클래스에 넣어서 모든 테스트 메소드에 일괄 적용할 수 있지만 @Rollback 애노테이션은 메소드 레벨에만 적용할 수 있다. 

테스트 클래스의 모든 메소드에 트랜잭션을 적용하면서 모든 트랜잭션이 롤백되지 않고 커밋되게 하려면 @TransactionConfiguration을 사용하면 롤백에 대한 공통 속성을 지정할 수 있다. 

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "/test-applicationContext.xml")
@Transactional
@TransactionConfiguration(defaultRollback=false)
public class UserServiceTest { // 롤백 여부에 대한 기본 설정과 트랜잭션 매니저 빈을 지정하는 데 사용할 수 있다.
                              // 디폴트 트랜잭션 매니저 아이디는 관례에 따라서 transactionManager로 되어 있다.
    @Test
    @Rollback // 메서드에서 디폴트 설정과 그 밖의 롤백 방법으로 재설정할 수 있다.
    public void add() throws SQLException {
        // ...
    }
}
```

효과적인 DB 테스트

- 테스트 내에서 트랜잭션을 제어할 수 있는 네 가지 애노테이션을 잘 활용하면 통합 테스트를 만들 때 편리함
- 단위 테스트와 통합 테스트는 클래스로 구분하는게 좋음
- DB가 사용되는 통합 테스트는 롤백 테스트로 만드는 것이 좋음