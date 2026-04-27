---
title: "Class Scanner"
date: 2026-04-27
categories: 
  - framework
tags:
  - java
---

## 목적

IoC container를 만들기 위해서 runtime 중에 annotation이 붙은 class를 확인하기 위한 class scanner를 구현하기

## 핵심 로직

### Annotation

Annotation은 태그와 같은 개념으로 실제로 로직이 돌아가는 것이 아니다.

```java
// class나 interface 위에서만 붙일 수 있게 함
@Target(ElementType.TYPE)
// Reflection API를 위해 runtime에서 이름표가 살아있도록 함
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {
    String value() default "";
}
```

### ClassLoader

Lazy loading: 모든 클래스 파일을 메모리에 올려두진 못하므로 Class.forName()나 객체를 생성할 때만 디스크에서 찾기 시작한다.

#### ClassLoader의 원칙

- delegation model : 로드 요청이 오면 일단 상위 계층의 classloader에게 위임
- visibility : parent는 child가 로드한 클래스를 볼 수 없지만 반대는 가능
- uniqueness : 한 번 로드한 것은 끝

#### ContextClassLoader

프레임워크는 나중에 외부 라이브러리로 묶여서 부모 계급의 ClassLoader에 의해 로드되지만 프레임워크를 사용하는 유저는 자식 계급인 Application ClassLoader에 의해 로드하므로 visibility 원칙으로 인해 ClassLoader에서 class를 찾을 수 없게 된다. 따라서 현재 실행 흐름을 담당하는 스레드를 통해서 ContextClassLoader를 가져와서 모든 클래스 로드할 수 있도록 한다.

#### Class.forName(String className)

여기서 className은 org.example.service.DummyService와 같은 (package path + class name)을 받아야 한다.

1. 문자열로 된 클래스 이름을 받아서 ClassLoader에게 파일을 찾도록 시킴
2. 파일을 찾아서 JVM의 메모리 영역에 올림
3. static 변수에 값을 할당하거나 static {…} 블록을 실행
4. 메모리에 올라간 클래스의 모든 정보를 Class<?>로 반환

## 전체 코드

### 메인코드

```java
package org.example;

import java.io.File;
import java.lang.annotation.Annotation;
import java.net.URL;
import java.util.ArrayList;
import java.util.List;

// 특정 경로를 주면 .class 파일 읽어오기
public class ClassScanner {

    private String parsePackageName(String packageName) throws Exception {
        String resourcePath = packageName.replace('.', '/');
        URL resource = Thread.currentThread().getContextClassLoader().getResource(resourcePath);

        if (resource == null) {
            throw new IllegalArgumentException("해당 패키지를 찾을 수 없습니다: " + packageName);
        }

        return resource.toURI().getPath();
    }

    private List<String> findClasses(String packageName, Boolean recursive) {
        List<String> classes = new ArrayList<>();
        try {
            // TODO: 나중에 .jar 파일로 패키징시켰을 때 실행이 안됨
            File[] files = new File(parsePackageName(packageName)).listFiles();
            if (files == null) return classes;
            for (File file : files) {
                if (recursive && file.isDirectory()) {
                    List<String> childClasses = findClasses(packageName + "." + file.getName(), recursive);
                    classes.addAll(childClasses);
                } else if (file.getName().endsWith(".class")) {
                    String simpleClassName = file.getName().substring(0, file.getName().length() - 6);
                    String fqcn = packageName + "." + simpleClassName;
                    classes.add(fqcn);
                }
            }
        } catch (Exception e) {
            // TODO: 명확한 Exception 처리가 필요
            e.printStackTrace();
        }
        return classes;
    }

    public List<Class<?>> scan(String packageName, Class<? extends Annotation> annotation, Boolean recursive) {
        List<Class<?>> classes = new ArrayList<>();

        try {
            List<String> fqcns = findClasses(packageName, recursive);
            ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
            for (String fqcn : fqcns) {
                Class<?> clazz = Class.forName(fqcn, false, contextClassLoader);
                if (clazz.isAnnotationPresent(annotation)) {
                    classes.add(clazz);
                }
            }
        } catch(Exception e) {
            System.out.println(e.getMessage());
        }

        return classes;
    }

    public ClassScanner() {
    }
}

```

### 테스트 코드

```java
@Test
@DisplayName("class scanner가 잘 동작하는지 확인하기 위한 테스트")
void applicationLevel() {
    ClassScanner classScanner = new ClassScanner();
    List<Class<?>> classes = classScanner.scan("org.example", Component.class, true);
    assertFalse(classes.isEmpty(), "스캔된 클래스가 비어있으면 안 됩니다.");
    System.out.println("성공적으로 찾아온 클래스 개수: " + classes.size());
}
```