---
title: "IoC Container 구현"
date: 2026-04-29
categories: 
  - framework
tags:
  - java
---

## 목적

- 클래스를 key로 가지고 객체를 value로 가지는 IoC Container 구현하기.
- Dependency Injection 구현하기.
- Dependency Injection 시 발생하는 Circular Dependency 문제 해결하기.  

## 핵심 로직

### ConcurrentHashMap을 활용한 이유

여러 HTTP 요청 thread를 받아 동시에 IoC container에 접근한다.

- HashMap
    
    Lazy Loading으로 runtime 중에 bean이 생성되면 심각한 동시성 이슈가 발생한다.
    
- Collections.synchronizedMap
    
    thread 안정성은 높아지지만 map 전체에 lock을 걸기 때문에 병목이 발생한다.
    
- ConcurrentHashMap
    
    내부적으로 lock을 분산시키기 때문에 여러 thread가 bean을 조회할 때 병목 없이 동시성을 보장한다.
    

동시성 보장 및 병목을 해결하기 위해 ConcurrentHashMap을 사용할 것이다.

※ Spring Boot에서는 Eager Loading을 사용하여 먼저 bean을 모두 생성하는 방식으로 동작하지만 Lazy Loading도 사용할 수 있도록 기능을 열어두었기 때문에 때문에 ConcurrentHashMap을 내부적으로 사용한다.

#### Lazy Loading vs Eager Loading

Lazy Loading은 요청을 받을 때 그 객체를 생성하는 방식이다. 반면 Eager Loading은 서버가 켜질 때 모든 객체를 미리 만드는 방식이다.

### Reflection 기술을 활용한 객체 생성

Reflection 기술을 통해서 runtime 도중에 Constructor를 가져와서 `newInstance()`를 통해서 객체를 생성한다.

### Constructor Injection을 선택한 이유

```java
@Target(ElementType.CONSTRUCTOR)
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {
    String value() default "";
}
```

일단 @autowired annotation이 붙어 있는 생성자를 찾는다. 그 다음 생성자의 parameter에 필요한 다른 dependency를 주입한다. 이는 topological sort를 재귀적으로 구현한 것이라고 할 수 있다. 필드 값에 바로 주입하는 방식(Field Injection)도 있으나 private 필드값을 강제로 수정한 것이기 때문에 Encapsulation 원칙이 깨진다고 볼 수 있다. 따라서 생성자의 파라미터에 dependency를 주입하는 방식으로 구현하였다.

※ topological sort란 먼저 수행해야 하는 작업이 존재할 때 어떤 순서대로 할지 결정하는 알고리즘으로 진입차수를 활용한 큐로 구현하는 방법과 재귀로 구현하는 방법이 있다.

### Set을 활용한 Circular Dependency 해결

만약 A ↔ B 상호적으로 dependency가 존재하고 circular dependency를 체크하지 않는 경우 `StackOverflowError`가 발생하여 서버가 다운된다. 따라서 이를 해결하기 위해 circular check를 해야 한다. 이를 해결하기 위한 방법으로 현재 생성 중인 객체의 타입을 확인하기 위해서 `Set`을 활용한다. 만약 재귀적으로 접근하다가 Set에 중복된 값을 추가하게 되어 false 값을 return하면 circular dependency가 발생했다고 볼 수 있다.

## 실제코드

### 메인코드

```java
package org.example;

import org.example.annotation.Autowired;

import java.lang.reflect.Constructor;
import java.lang.reflect.Parameter;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

// todo: String 으로도 객체를 찾을 수 있도록 수정
// todo: NoArgsConstructor 도 사용할 수 있도록 수정
// todo: Error 로그에 Circular dependency에 포함되지 않은 class도 출력되는 문제 발생
public class IoCContainer {
    private Map<Class<?>, Object> container = new ConcurrentHashMap<>();
    private Set<Class<?>> circularDependencyCheck = new HashSet<>();

    public void addBean(Object bean) {
        container.put(bean.getClass(), bean);
    }

    public <T> T getBean(Class<T> clazz) {
        return clazz.cast(container.get(clazz));
    }

    private Constructor<?> getConstructor(Class<?> clazz) {
        Constructor<?>[] constructors = clazz.getDeclaredConstructors();

        List<Constructor<?>> targetConstructors = Arrays.stream(constructors)
                .filter(c -> c.isAnnotationPresent(Autowired.class))
                .toList();

        if (targetConstructors.size() != 1) {
            throw new RuntimeException("Autowired error: " + targetConstructors.size());
        }

        return targetConstructors.getFirst();
    }

    private Object getObjectWithConstructor(Class<?> clazz) {
        try{
            boolean notCircular = circularDependencyCheck.add(clazz);
            if (!notCircular) {
                throw new RuntimeException("Circular dependency 발생: " + circularDependencyCheck.stream().toList());
            }

            Constructor<?> constructor = getConstructor(clazz);
            Object bean;
            Parameter[] parameters = constructor.getParameters();
            if (parameters.length == 0) {
                bean = constructor.newInstance();
                this.addBean(bean);
                return bean;
            }
            Object[] initArgs = new Object[parameters.length];
            for (int i = 0; i < parameters.length; i++) {
                Object instance = this.getBean(parameters[i].getType());
                if (instance == null) {
                    instance = getObjectWithConstructor(parameters[i].getType());
                }
                initArgs[i] = instance;
            }
            bean = constructor.newInstance(initArgs);
            this.addBean(bean);
            return bean;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public IoCContainer(List<Class<?>> classes) {
        for (Class<?> clazz : classes) {
            if (container.containsKey(clazz)) {
                continue;
            }
            try {
                circularDependencyCheck.clear();
                getObjectWithConstructor(clazz);
            } catch (Exception e) {
                throw new RuntimeException("빈 초기화 실패: " + clazz.getName() + "\n" + e.getMessage());
            }
        }
    }
}

```

### 테스트 코드

```java
package org.example.controller;

import org.example.annotation.Autowired;
import org.example.annotation.Component;
import org.example.service.DummyService;

@Component
public class DummyController {
    DummyService dummyService;
    CircularController circularController;

    public void doSomething() {
        dummyService.doSomething();
    }

    @Autowired
    public DummyController(DummyService dummyService, CircularController circularController) {
        this.dummyService = dummyService;
        this.circularController = circularController;
    }
}
```

```java
package org.example.controller;

import org.example.annotation.Autowired;
import org.example.annotation.Component;

@Component
public class CircularController {
    private DummyController dummyController;

    @Autowired
    public CircularController(DummyController dummyController) {
        this.dummyController = dummyController;
    }

    public CircularController() {
    }
}
```

```java
@Test
@DisplayName("DI 동작잘하는지 테스트")
public void DependencyInjectionTest() {
    ClassScanner classScanner = new ClassScanner();
    List<Class<?>> classes = classScanner.scan("org.example", Component.class, true);
    IoCContainer container = new IoCContainer(classes);

    container.getBean(DummyController.class).doSomething();
}

@Test
@DisplayName("Circular Dependency 테스트")
public void CircularDependencyTest() {
    ClassScanner classScanner = new ClassScanner();
    List<Class<?>> classes = classScanner.scan("org.example", Component.class, true);

    assertThatThrownBy(() -> new IoCContainer(classes))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("Circular dependency 발생");
}
```