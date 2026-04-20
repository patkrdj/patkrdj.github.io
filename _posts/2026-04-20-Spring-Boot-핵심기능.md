---
title: "Spring Boot 핵심기능"
date: 2024-04-20
categories: 
  - spring boot
tags:
  - java
---

# 핵심 기능

- DI(Dependency Injection) 및 IoC(Inversion of Control)
- AOP(Aspect-Oriented Programming)
- PSA(Portable Service Abstraction)

## DI 및 IoC

IoC(Inversion of Control)은 소프트웨어 디자인 원칙으로써 해석하자면 “제어의 역전”이라고 말할 수 있다. 이는 코드의 제어권을 외부를 역전시킨다는 의미이다. 여기서 코드의 제어권은 두 가지 관점에서 볼 수 있다.

- 객체의 생성과 생명 주기 제어권
    
    Spring Boot에서는 이러한 클래스가 있다는 것을 `@Service` 와 같은 annotation을 통해서 Bean으로 등록하고 `ApplicationContext` 라는 거대한 container를 통해서 관리한다.
    
- 프로그램의 전체적인 흐름 제어권
    
    Spring Boot에서는 전체적인 요청 처리(스케줄링, HTTP 요청)을 수행하다가 사용자의 요청이 들어왔을 때, `DispatcherServlet` 가 개발자가 작성한 코드(`@Controller`)로 라우팅을 수행한다.
    

이를 흔히 **할리우드 원칙**이라고 부른다. “Don’t call us, we’ll call you”

DI(Dependency Injection)은 한 객체가 다른 객체가 필요할 때 그 객체를 주입함으로써 동작하는 방식으로 Spring Boot에서 IoC를 구현한 기술을 이른다. 따라서 IoC와 DI의 관계는 IoC는 원칙이고 DI는 기술이라고 볼 수 있고, DI 기술을 사용하지 않더라도 IoC는 만족할 수 있는 부분집합의 관계를 가진다.

```java
@Service
@RequiredArgsConstructor
public class BookService {
    public final BookRepository bookRepository;
	
		//@Autowired
    @Transactional(readOnly = true)
    public List<Book> getBooks() {
        return bookRepository.findAll();
    }
}
```

이러한 코드를 보았을 때 bookRepository의 객체를 생성하지 않았음에도 bookRepsitory를 사용할 수 있음을 알 수 있다. `@Autowired`는 DI를 위한 annotation으로 프레임워크가 객체를 주입했으나 Spring 4.3부터 생성자가 1개인 클래스는 `@Autowired`를 적지 않더라도 알아서 생략된 것으로 알고 DI를 주입하도록 변했다. `@Service`는 Bean 생성을 위한 annotation으로 DI가 아닌 IoC를 구현한 다른 기술적 장치라고 할 수 있다.

### Actuator를 활용한 Bean 확인하기

```java
// build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

```java
// application.properties
management.endpoints.web.exposure.include=beans
```

이런 식으로 코드를 추가하고 난 다음 run을 시키고 “http://localhost:8080/actuator/beans” 이 주소에 접속하면 모든 Bean들이 json 파일로 저장되어 있는 것을 볼 수 있다.

## AOP

AOP(Aspect-Oriented Programming)은 로깅, 보안, 트랜잭션, 예외 처리 등의 모든 공통적인 코드를 비즈니스 로직에서 분리시키는 기술이다.

```java
@Service
@RequiredArgsConstructor
public class BookService {
    public final BookRepository bookRepository;
	
    @Transactional(readOnly = true)
    @LogExecutionTime
    public List<Book> getBooks() {
        return bookRepository.findAll();
    }
}
```

위의 코드를 보았을 때 `@Transactional` 과 `@LogExecutionTIme` 등의 annotation을 통해서 트랜잭션과 로깅에 대한 공통적인 코드를 비즈니스 코드에서 분리하고 AOP 모듈에서 관리하고 있는 것을 확인할 수 있다.

### AOP 모듈의 구성요소

1. Target : 핵심 비즈니스 코드가 있는 진짜 객체(`BookService`)
2. Aspect : 공통적으로 쓰이는 부가 기능 모듈 자체(`@LogAspect`, `@TransactionAspect`)
3. Advice : 부가 기능을 언제 쓸 것인지(`@Before`, `@After`)
4. Pointcut : 부가 기능을 어떤 메소드에 적용할 것인가

```java
package com.example.aop;

import org.aspectj.lang.*;
import org.springframework.stereotype.Component;

@Aspect  // 2. Aspect
@Component  // 스프링 컨테이너에 빈으로 등록되어야 프록시 공장이 작동함.
public class CoffeeLoggingAspect {

    // 4. Pointcut
    // com.example.service 패키지 아래의 BookService 클래스가 가진 모든 메서드(* (..))에 적용
    @Pointcut("execution(* com.example.service.BookService.*(..))")
    private void targetMethods() {}

    // 🌟 3. Advice
    // @Around: 타겟 메서드의 실행 '전과 후' 모두 실행하겠다.
    @Around("targetMethods()") // 위에서 정의한 Pointcut을 가리킴
    public Object checkBookProcess(ProceedingJoinPoint joinPoint) throws Throwable {

        Object[] args = joinPoint.getArgs(); // 사용자가 넘긴 파라미터를 몰래 가로챈다.
        String book = (String) args[0];
        System.out.println("[프록시] " + book + "파리미터 확인");

        // 진짜 객체를 실행(Delegate 개념)
        Object result = joinPoint.proceed();

        System.out.println("<<< [프록시] 커피가 완성되었습니다. 안녕히 가세요!");
        
        return result;
    }
}
```

### Proxy 기술

Spring IoC 컨테이너가 Bean을 만들 때 타겟 객체에 AOP가 적용되어야 한다면 진짜 객체 대신 진짜와 똑같이 생긴 가짜 객체(Proxy)를 덮어씌워서 컨테이너에 등록한다.

※ Proxy는 가짜 객체라고 하지만 사실상 진짜 객체를 안에 가지고 있다.

```java
// 스프링이 몰래 만든 가짜 객체 (Proxy)
public class BookServiceProxy extends BookService {

    private final BookService realService;

    @Override
    public List<Book> getBooks() {
        System.out.println("트랜잭션 시작"); // AOP
			   
        List<Book> result = realService.getBooks(); // 진짜 객체 호출!
        
        System.out.println("트랜잭션 커밋"); // AOP
        
        return result;
    }
}
```

위의 코드는 이해하기 쉽도록 만든 Pseudocode이다. 실제로는 Spring이 런타임 시점에 바로 바이트코드 형태로 메모리에 찍혀나오게 된다. 이 때 Spring이 사용하는 기술이 CGLIBI(Code Generation Library)이다.

## PSA

PSA(Portable Service Abstraction)는 데이터베이스나 웹 서버등이 변경되어도 비즈니스 로직은 변하지 않는다는 원칙이다. 

```java
@Service
@RequiredArgsConstructor
public class BookService {
    public final BookRepository bookRepository;
	
    @Transactional(readOnly = true)
    @LogExecutionTime
    public List<Book> getBooks() {
        return bookRepository.findAll();
    }
}
```

이 코드에서 데이터베이스가 변경되더라도 `@Transactional` 로 감싸진 내부 비즈니스 코드는 변하지 않는다. Spring 내부의 `PlatformTransactionManager` 라는 거대한 PSA(추상화 인터페이스)가 세팅한 기술에 맞춰서 데이터베이스 커밋을 날린다.

```java
@GetMapping("/books")
public List<Book> getBooks() { ... }
```

웹 서버도 마찬가지로 웹 서버가 변경되더라도 `@GetMapping("/books")`를 그대로 사용할 수 있다.

# 마무리

다음 내용으로는 JPA의 원리에 대해서 소개할 것이다.