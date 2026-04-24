---
title: "Java Framework 구현 목차"
date: 2024-04-24
categories: 
  - framework
tags:
  - java
---

## 목적

Spring Boot의 DI, AOP, PSA의 핵심 기능을 직접 구현하면서 Java 및 프레임워크의 이해도를 높이는 것이 목표이다. 아래의 체크리스트를 차례대로 구현하여 최종적으로 프레임워크를 완료할 예정이다.

## TODO List

### IoC 컨테이너 및 DI 구현

- @Component annotation 구현하기
- Classpath Scanner 구현하기: 특정 경로를 주면 `.class` 파일 읽어오기
- IoC 컨테이너 구현하기: annotation이 붙은 클래스를 `Reflection`으로 찾아 Map 형태로 저장하기
- @Autowired annotation 구현하기: 컨테이너를 순회하며 의존성 주입(Circular Dependency 방지)

### AOP 및 PSA 구현

- @Transactional 구현하기: `JDK Dynamic Proxy` 또는 `CGLIB` 사용
- @Log 구현하기
- PSA를 적용하기 위해 인터페이스로 설계하기

### Web & MVC

- `Embedded Tomcat`를 라이브러리에 추가해서 서버 띄우기
- 표준 Servlet 만들어서 HTTP 요청 받기
- @Controller 구현하기
- @GetMapping 구현하기

### 데이터 엑세스와 자동 연결화

- 커넥션 풀(Connection Pool) 구현
- JdbcTemplate 템플릿 메서드 패턴 구현: Connection 등의 자원 해제(try-catch-finally)의 반복을 없애주는 헬퍼 클래스 만들기
- RowMapper 인터페이스 도입: SQL 쿼리 결과를 자바 객체로 자동으로 매핑해주는 콜백 인터페이스 구현하기
- 내장 톰캣(Embedded Tomcat) 흉내내기
- 로고(Banner) 출력
- 데모 애플리케이션 작성