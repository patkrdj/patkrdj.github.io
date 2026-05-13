---
title: "Reflection 기술을 통한 Proxy 구현"
date: 2026-05-13
categories: 
  - framework
tags:
  - java
---

## Proxy 이해하기

프록시는 기존 주체를 대신하는 대리인(Surrogate) 역할을 하며, 실제 로직 처리는 주체에게 위임(Delegation)하는 객체를 말한다. 프록시를 두게 되면 클라이언트가 주체에게 직접 접근하는 것을 제어할 수 있을 뿐만 아니라, 주체의 기존 핵심 코드를 전혀 건드리지 않고도 메서드 실행 전후로 부가적인 코드(로깅, 트랜잭션 등)를 삽입할 수 있다.

이를 통해 비즈니스 로직과 공통적인 코드를 분리함으로써 Java Framework가 AOP를 구현할 수 있게 된다. runtime에 프록시를 자동으로 생성하는 동적 프록시 기술을 활용할 것이다.

## JDK Dynamic Proxy

새로운 proxy를 생성하기 위해 ClassLoader, Interface, Invocation Handler가 필요하다.

- ClassLoader : 새로운 proxy를 적재할 때, 원본 객체와 동일한 타입으로 인식되도록 하기 위해 원본의 class loader를 사용한다.
    
    ※ metaspace: JVM에서 메타 데이터를 저장하기 위해서 class loader에 할당하는 동적 메모리 영역
    
- Interface : JDK Dynamic Proxy는 바이트코드를 생성할 때 Proxy를 상속받도록 강제되어 있다. 하지만 Java의 특성 상 단일 상속만을 지원하기 때문에 다중 구현이 가능한 Interface를 사용하는 방법 밖에 없다.
- Invocation Handler : 프록시의 주된 역할을 하는 부분으로 Interface method인 invoke를 구현한다. 이를 통해 method invoke 전후로 프록시에 해당하는 로직을 추가한다.

### 코드

```java
package org.example;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class TransactionalProxyBuilder {
    public static Object wrapWithProxy(Object target, TransactionManager txManager) {
        InvocationHandler handler = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                if (method.isAnnotationPresent(Transactional.class) || 
                    target.getClass().getAnnotation(Transactional.class) != null) {
                    
                    TransactionManager.begin();
                    try {
                        Object result = method.invoke(target, args); 
		                    TransactionManager.commit();
                        return result;
                    } catch (Exception e) {
                        TransactionManager.rollback();
                        throw e;
                    }
                }
                return method.invoke(target, args);
            }
        };

        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            handler
        );
    }
}
```

### 문제점

결국 JDK Dynamic Proxy를 사용하는 경우, IoC Container에 추가하려는 proxy 객체들은 모두 Interface를 상속한 상태여야 한다. 따라서 이를 해결하기 위해서는 원본 클래스의 인터페이스를 추출해서 주입받게 하거나 proxy를 생성할 때 proxy를 상속받지 않고 그 class 바로 상속하는 코드를 생성하는 proxy library로 변경해야 한다.
이것의 근본적인 원인은 2가지가 있다. 첫 번째는 Invocation Handler 객체가 무조건 필요했기 때문에 이를 강제하기 위해서이고, 두 번째는 Java 설계자가 reflection 기술 자체가 runtime 때에 바이트코드를 동적으로 생성하기 때문에 아무 클래스나 상속해서 호출하는 것이 위험하다고 판단했기 때문이다.

## CGLIB

CGLIB과 같은 바이트코드 조작 라이브러리는 자식 클래스 내부에 MethodInterceptor를 저장할 수 있는 전용 라우팅 로직을 하드코딩해서 넣었고, 바이트코드를 정밀하게 조작할 수 있는 기술을 통해 안정적으로 자식 클래스를 만들 수 있어졌기 때문에 기존의 Proxy 클래스를 상속받지 않아도 되도록 하였다.
CGLIB을 구현하기 위해서는 Interceptor와 Enhancer가 필요하다

- Interceptor : proxy를 위한 규칙, 나중에 Enhancer의 callback 함수로 사용된다.
- Enhancer : 바이트코드 조작을 담당하는 부분

### 코드

**Interceptor**

```java
package org.example;

import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;
import java.lang.reflect.Method;

public class TransactionalMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        TransactionManager.begin();
        try {
            Object result = methodProxy.invokeSuper(o, objects);
            TransactionManager.commit();
            return result;
        } catch (Exception e) {
            TransactionManager.rollback();
            throw e;
        }
    }
}
```

**Enhancer**

```java
package org.example;

import net.sf.cglib.proxy.Enhancer;

public class TransactionalProxyBuilder {
    public static Object wrapWithProxy(Class<?> targetClass) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(targetClass);
        enhancer.setCallback(new TransactionalMethodInterceptor());
        return enhancer.create();
    }
}
```

### 문제점

`Caused by: java.lang.reflect.InaccessibleObjectException: Unable to make protected final java.lang.Class`

위의 코드를 구현하고 나서 이러한 에러 로그가 뜨게 된다. 이 로그의 의미는 리플렉션으로 `java.lang.Class`에 접근하려고 하였으나 실패하였다는 의미이다. 이는 CGLIB 라이브러리 자체의 문제로, CGLIB은 라이브러리 내에서 `setAccessible(true)` 를 통해서 method에 강제로 접근하는 식으로 동작한다. Java 16부터는 JDK 내부 API 정책이 변경되면서 `java.lang` 패키지에 대한 리플렉션 접근 자체를 차단하도록 바뀌었다. 하지만 CGLIB은 2019년 이후로 업데이트가 멈춘 구형 프로젝트이기 때문에 최신 Java를 사용하면 위의 에러가 발생하게 된다.

※ Spring Boot에서는 원본 CGLIB을 사용하지 않고 코드를 직접 복사한 다음 자바 버전에 맞게 내부적으로 뜯어고친 spring-cglib을 사용한다.

## ByteBuddy

현대 Java에서는 주로 ByteBuddy라는 reflection 라이브러리를 사용한다. CGLIB은 `setAccessible(true)`코드를 계속 사용하는 반면 ByteBuddy는 내부적으로 `java.lang.invoke.MethodHandles.Lookup.defineClass()`라는 API를 사용하기 때문에 에러가 발생하지 않는다. 또한 만약 이 방법으로도 해결하기 어려운 상황이라면 `sun.misc.Unsafe` 라는 low-level API를 사용한다. 여기서 ByteBuddy의 핵심적인 부분은 Java의 환경에 따라서 기존 reflection 방식을 사용할지, `Lookup.defineClass`를 사용할지, `Unsafe`를 사용할지 알아서 결정해준다.

### 코드

**Interceptor**

```java
package org.example;

import net.bytebuddy.implementation.bind.annotation.RuntimeType;
import net.bytebuddy.implementation.bind.annotation.SuperCall;

import java.util.concurrent.Callable;

public class TransactionalMethodInterceptor {
    @RuntimeType
    public Object intercept(@SuperCall Callable<?> superMethod) throws Exception {
        TransactionManager.begin();
        try {
            Object result = superMethod.call();
            TransactionManager.commit();
            return result;
        } catch (Exception e) {
            TransactionManager.rollback();
            throw e;
        }
    }
}
```

**ByteBuddy**

```java
package org.example;

import net.bytebuddy.ByteBuddy;
import net.bytebuddy.implementation.MethodDelegation;
import net.bytebuddy.matcher.ElementMatchers;

public class TransactionalProxyBuilder {
    public static <T> T wrapWithProxy(Class<T> targetClass) throws Exception {
        return new ByteBuddy()
                .subclass(targetClass)
                .method(ElementMatchers.any())
                .intercept(MethodDelegation.to(new TransactionalMethodInterceptor()))
                .make()
                .load(targetClass.getClassLoader())
                .getLoaded()
                .getDeclaredConstructor()
                .newInstance();
    }
}
```

**IoC Container** : 기존 객체 생성 방법이 아닌 ByteBuddy를 활용하면서 코드가 변경됨

```java
private Object getObjectWithConstructor(Class<?> clazz) {
    try{
        boolean notCircular = circularDependencyCheck.add(clazz);
        if (!notCircular) {
            throw new RuntimeException("Circular dependency 발생: " + circularDependencyCheck.stream().toList());
        }

        Constructor<?> constructor = getConstructor(clazz);
        Parameter[] parameters = constructor.getParameters();
        if (parameters.length == 0) {
            Object proxy = TransactionalProxyBuilder.wrapWithProxy(clazz);
            Field[] fields = clazz.getDeclaredFields();
            
            for (Field field : fields) {
                field.setAccessible(true);
                Object instance = this.getBean(field.getType());
                if (instance == null) {
                     instance = this.getObjectWithConstructor(field.getType());
                }
                field.set(proxy, instance);
            }
            this.addBean(clazz, proxy);
            circularDependencyCheck.remove(clazz);
            return proxy;
            
        } else {
            throw new RuntimeException("현재 프록시는 기본 생성자(NoArgsConstructor)가 있는 클래스만 지원합니다: " + clazz.getName());
        }
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

### 문제점

결국 ByteBuddy는 객체를 생성하고 proxy로 감싸는 방식이 아닌 바로 proxy로 감싼 객체를 만드는 방식을 동작한다. 이 과정에서 getDeclaredConstructor().newInstance()를 사용하기 때문에 argument가 없는 생성자가 무조건적으로 필요하게 된다. 이를 해결하기 위해서는 proxy로 감싸야하는 모든 클래스가 NoArgsConstructor를 필요로 하게 된다. 다른 방법으로는 `Objenesis`라는 라이브러리를 추가하는 방법이 있다. 이를 활용하면 Java의 생성자 규칙을 무시하고 메모리 상에 바로 객체를 할당할 수 있게 한다. 추후 `Objenesis`를 활용하여 프레임워크를 확장할 예정이다.

## 테스트 코드

### 코드

```java
@Test
@DisplayName("DI 동작잘하는지 테스트")
public void DependencyInjectionTest() {
    ClassScanner classScanner = new ClassScanner();
    List<Class<?>> classes = classScanner.scan("org.example", Component.class, true);
    IoCContainer container = new IoCContainer(classes);

    container.getBean(DummyController.class).doSomething();
}
```