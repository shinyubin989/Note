### 목차
1. [문제상황](#문제상황)
2. [원인](#원인)
    1. [한 줄 정리](#한-줄-정리)
3. [원인 이해를 위한 지식](#원인-이해를-위한-지식)
    1. [리플렉션](#리플렉션)
    2. [인터페이스 기반 프록시 vs 클래스 기반 프록시](#인터페이스-기반-프록시-vs-클래스-기반-프록시)
    3. [IoC 컨테이너와 AOP Proxy의 관계](#ioc-컨테이너와-aop-proxy의-관계)
    4. [두 가지 AOP Proxy는 어떤 상황에 생성하게될까?](#두-가지-aop-proxy는-어떤-상황에-생성하게될까)
    5. [왜 프록시를 사용하나?](#왜-프록시를-사용하나)
    6. [JDK dynamic proxy를 쓸지, CGLib proxy를 쓸지 어떻게 결정할까?](#jdk-dynamic-proxy를-쓸지-cglib-proxy를-쓸지-어떻게-결정할까)
    7. [JDK Dynamic Proxy의 Proxy객체 생성방법](#jdk-dynamic-proxy의-proxy객체-생성방법)
    8. [CGLib의 Proxy객체 생성방법](#cglib의-proxy객체-생성방법)
    9. [두 기술을 함께 사용할 때 부가 기능을 적용하기 위해 JDK 동적 프록시가 제공하는 InvocationHandler와 CGLIB가 제공하는 MethodInterceptor를 각각 중복으로 따로 만들어야할까?](#두-기술을-함께-사용할-때-부가-기능을-적용하기-위해-jdk-동적-프록시가-제공하는-invocationhandler와-cglib가-제공하는-methodinterceptor를-각각-중복으로-따로-만들어야할까)
4. [해결](#해결)
5. [번외](#번외)


![image](https://github.com/shinyubin989/note/assets/69676101/24d90d66-4df6-46f8-987e-d1dbd1ecb8e6)


![image](https://github.com/shinyubin989/note/assets/69676101/ebcfa4ec-bb52-44d1-bc12-07208aed15cd)


## 문제상황

인터페이스를 구현하지 않은 클래스를 빈으로 등록하였다. 그리고 클래스 내에는 @Transactional (from jakarta)이 붙은 메소드가 존재한다.

이를 컴파일할 시에 `Could not generate CGLIB subclass of class 'ConsumerAuthProvider': Common causes of this problem include using a final class or a non-visible class`, 즉 클래스의 CGLIB를 생성할 수 없다는 의미의 문구로 컴파일 에러가 발생한다.

## 원인

### **한 줄 정리**

→ Spring AOP의 **클래스 기반 프록시**를 사용하려는 경우 **클래스가 상속 가능**해야 한다.

## 원인 이해를 위한 지식

### **리플렉션**

- 기본적으로 리플렉션은 메타정보를 얻고, 이를 통해 동적으로(런타임에) 내가 원하는 방향으로 바꾸는게 가능하다. (동적으로 실행할 메소드를 변경하거나 … 등등)

### **인터페이스 기반 프록시 vs 클래스 기반 프록시**

- 인터페이스가 없어도 클래스 기반으로 프록시를 생성할 수 있다.
- 클래스 기반 프록시는 해당 클래스에만 적용할 수 있다. 인터페이스 기반 프록시는 인터페이스만 같으면 모든 곳에 적용할 수 있다.
- 클래스 기반 프록시는 상속을 사용하기 때문에 몇가지 제약이 있다.
    - 부모 클래스의 생성자를 호출해야 한다.
    - 클래스에 final 키워드가 붙으면 상속이 불가능하다.
    - 메서드에 final 키워드가 붙으면 해당 메서드를 오버라이딩 할 수 없다.
- 이렇게 보면 인터페이스 기반의 프록시가 더 좋아보인다. 맞다. 인터페이스 기반의 프록시는 상속이라는 제약에서 자유롭다. 프로그래밍 관점에서도 인터페이스를 사용하는 것이 역할과 구현을 명확하게 나누기 때문에 더 좋다.
- 인터페이스 기반 프록시의 단점은 인터페이스가 필요하다는 그 자체이다. 인터페이스가 없으면 인터페이스 기반 프록시를 만들 수 없다.

### IoC 컨테이너와 AOP Proxy의 관계

- Spring AOP는 **Proxy의 메커니즘**을 기반으로 AOP Proxy를 제공하고 있다.
    
    ![image](https://github.com/shinyubin989/note/assets/69676101/72970db7-2fd8-49a5-8f3e-ee618f5924f1)

    
- 그림처럼 Spring AOP는 사용자의 **특정 호출 시점**에 IoC 컨테이너에 의해 **AOP를 할 수 있는 Proxy Bean을 생성**해준다.
- 동적으로 생성된 Proxy Bean은 타깃의 메소드가 호출되는 시점에 **부가기능을 추가할 메소드를 자체적으로 판단하고 가로채어 부가기능을 주입**해주는데, 이처럼 호출 시점에 동적으로 위빙을 한다 하여 런타임 위빙(Runtime Weaving)이라 한다.
- 따라서 Spring AOP는 런타임 위빙의 방식을 기반으로 하고 있으며, Spring에선 런타임 위빙을 할 수 있도록 상황에 따라 JDK Dynamic Proxy와 CGLIB 방식을 통해 Proxy Bean을 생성을 해준다.

### 두 가지 AOP Proxy는 어떤 상황에 생성하게될까

- Spring은 AOP Proxy를 생성하는 과정에서 자체 검증 로직을 통해 타깃의 인터페이스 유무를 판단한다.
    
    ![image](https://github.com/shinyubin989/note/assets/69676101/2ec10603-ed81-4417-abc2-0f1bf916360c)

    
- 만약 타깃이 하나 이상의 인터페이스를 구현하고 있는 클래스라면 JDK Dynamic Proxy의 방식으로 생성되고 인터페이스를 구현하지 않은 클래스라면 CGLIB의 방식으로 AOP 프록시를 생성한다.

### 왜 프록시를 사용하나

- 프록시 객체란
    - 프록시 객체는 원래 객체를 감싸고 있는 객체로, 원래 객체와 타입은 동일하다.
    - 프록시 객체가 원래 객체를 감싸서 client의 요청을 처리하게 하는 패턴이다.
        - 접근 권한을 부여할 수 있다.
        - 부가 기능을 추가할 수 있다.
- `@Transactional` 이 붙은 메소드는 왜 프록시가 필요할까
    - AOP의 proxy로 transaction의 `begin`과 `commit`을 메인 로직 앞 뒤로 수행해줘야 한다.
    - @Transactional 이 붙은 메서드가 호출되기 전 begin을 호출하고, 메서드가 종료되고 commit을 호출한다.
    - 이 때 Spring AOP는 기본적으로 proxy패턴을 사용한다.
    - 트랜잭션이 붙은 메소드는 아래와같이 프록시 기술이 적용될 것이다.
        
        ```java
        public void methodA(){
        	  EntityTransaction tx = em.getTransaction(); // aop에 의해 추가된 코드
        	  tx.begin(); // aop에 의해 추가된 코드
        	  
        		//
        	  // 기존 로직
        		//
        	  
        	  tx.commit(); // aop에 의해 추가된 코드
        }
        ```
        

### JDK dynamic proxy를 쓸지 CGLib proxy를 쓸지 어떻게 결정할까

- 스프링은 유사한 구체적인 기술들이 있을 때, 그것들을 통합해서 일관성 있게 접근할 수 있고, 더욱 편리하게 사용할 수 있는 추상화된 기술을 제공한다.
- 스프링은 동적 프록시를 통합해서 편리하게 만들어주는 프록시 팩토리(`ProxyFactory`)라는 기능을 제공한다.
- 이전에는 상황에 따라서 JDK 동적 프록시를 사용하거나 CGLIB를 사용해야 했다면, 이제는 이 프록시팩토리 하나로 편리하게 동적 프록시를 생성할 수 있다.
- 프록시 팩토리는 인터페이스가 있으면 JDK 동적 프록시를 사용하고, 구체 클래스만 있다면 CGLIB를 사용한다. 그리고 이 설정을 변경할 수도 있다.
    
    ![image](https://github.com/shinyubin989/note/assets/69676101/eac54769-27db-42e4-b860-6d26dd1101d7)

    
    - 프록시 팩토리의 서비스 추상화 덕분에 구체적인 CGLIB, JDK 동적 프록시 기술에 의존하지 않고, 매우 편리하게 동적 프록시를 생성할 수 있다.
    - 프록시의 부가 기능 로직도 특정 기술에 종속적이지 않게 Advice 하나로 편리하게 사용할 수 있다.
        - 이것은 프록시 팩토리가 내부에서 JDK 동적 프록시인 경우 InvocationHandler가 Advice를 호출하도록 개발해두고, CGLIB인 경우 MethodInterceptor가 Advice를 호출하도록 기능을 개발해두었기 때문이다.

### JDK Dynamic Proxy의 Proxy객체 생성방법

```java
Object proxy = Proxy.newProxyInstance(ClassLoader       // 클래스로더
                                    , Class<?>[]        // 타깃의 인터페이스
                                    , InvocationHandler // 타깃의 정보가 포함된 Handler
```

### CGLib의 Proxy객체 생성방법

```java
Enhancer enhancer = new Enhancer();
         enhancer.setSuperclass(MemberService.class); // 타깃 클래스
         enhancer.setCallback(MethodInterceptor);     // Handler
Object proxy = enhancer.create(); // Proxy 생성
```

### **두 기술을 함께 사용할 때 부가 기능을 적용하기 위해 JDK 동적 프록시가 제공하는 InvocationHandler와 CGLIB가 제공하는 MethodInterceptor를 각각 중복으로 따로 만들어야할까**

![image](https://github.com/shinyubin989/note/assets/69676101/29bc088a-6fa3-4783-aed8-c9edcdc12cba)


- 스프링은 이 문제를 해결하기 위해 부가 기능을 적용할 때 **Advice**라는 새로운 개념을 도입했다.
- 개발자는 InvocationHandler나 MethodInterceptor를 신경쓰지 않고, Advice만 만들면 된다.
- 결과적으로 InvocationHandler나 MethodInterceptor는 Advice 를 호출하게 된다.
- 프록시 팩토리를 사용하면 Advice를 호출하는 전용 InvocationHandler, MethodInterceptor를 내부에서 사용한다.
- MethodIntercepter는 Intercepter를 상속하고, Intercepter는 Advice 인터페이스를 상속한다.
    
    ```java
    /**
    	* 위 그림의 cglib proxy부분(밑부분) 화살표를 함께 따라가면서 코드를 읽어보자.
    **/
    
    public class TimeAdvice implements MethodInterceptor {
    	@Override
    	public Object invoke(MethodInvocation invocation) throws Throwable {
    		long startTime = System.currentTimeMillis();
    
    		Object result = invocation.proceed();
    
    		long endTime = System.currentTimeMillis();
    		long resultTime = endTime - startTime; 
    		return result;
    	}
    }
    
    public class ProxyFactoryTest {
      @Test
    	@DisplayName("인터페이스가 있으면 JDK 동적 프록시 사용") 
    	void interfaceProxy() {
    		ServiceInterface target = new ServiceImpl();
        ProxyFactory proxyFactory = new ProxyFactory(target);
        proxyFactory.addAdvice(new TimeAdvice());
        ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
    
        proxy.save();
    
        assertThat(AopUtils.isAopProxy(proxy)).isTrue();
        assertThat(AopUtils.isJdkDynamicProxy(proxy)).isTrue();
        assertThat(AopUtils.isCglibProxy(proxy)).isFalse();
    	}
    }
    ```
    

## 해결

class level에 open키워드를 달아 해결했다.

클래스에 `@Transactional` 을 달면, 설정한 all-open 플러그인에 의해 open키워드를 달지 않아도 되지만, 이번 경우에는 한 클래스의 모든 메소드가 트랜잭션에 의해 처리되길 원치 않았기 때문에 클래스에 open을 다는 것으로 하나의 메소드만 트랜잭션 처리가 되도록 하여 마무리 지었다.

## 번외

Intellij 2021년이 아닌 2023년 버전을 사용하면 login메소드에 빨간줄로 open키워드를 사용하라고 안내해준다. 당시 2021년버전은 이를 알려주지 않아 컴파일시까지 문제가 된다는 걸 몰랐다.
- Intellij 2021년 버전 - 빨간줄이 안뜬다
<img src="https://github.com/shinyubin989/note/assets/69676101/cce93586-9889-46d7-a83b-3de2e320eb26" width=50%/>


- Intellij 2023년 버전 - 빨간줄로 알려준다
<img src="https://github.com/shinyubin989/note/assets/69676101/31c46278-9f41-463f-9125-d2e22a918cf8" width=50% />


<br><br>

**reference : 인프런 - 스프링 핵심 원리 고급편(김영한)**
