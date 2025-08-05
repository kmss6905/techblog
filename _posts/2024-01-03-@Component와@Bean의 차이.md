---
layout: post
title: "@Component와 @Bean의 차이"
tags:
  - spring
  - bean
  - component
  - configuration
  - 스프링
  - 자바
  - DI
  - IoC
date: "2024-01-03"
---
> 예제에서 사용한 모든 코드는 [Github Repository](https://github.com/kmss6905/blog/tree/main/ioc) 에 있다.

# 스프링 빈 등록 방법: @Configuration, @Component, @Bean

스프링으로 개발할 때 필요한 객체는 스프링 컨테이너에서 관리하는 빈으로 등록하여 사용한다.

이때 @Configuration 설정과 함께 원하는 의존관계를 만족하는 인스턴스를 반환하는 메서드를 만들고, 그 위에 @Bean 애노테이션을 선언한다. 또는 필요한 클래스 위에 @Component를 선언하여 사용한다.

주로 @Configuration과 함께 @Bean을 사용한다. 왜 @Component는 같이 사용하지 않을까? 그리고 @Bean과 @Component는 어떤 차이가 있을까.

## @Bean과 @Configuration의 역할과 특징

@Bean은 메소드 수준에서 사용할 수 있는 애노테이션이다. 그리고 @Configuration과 @Bean이 같이 선언되었을 때, 선언된 메서드는 스프링 컨테이너에 의해 관리될 빈을 생성한다.

이렇게 @Bean이 붙은 메서드에서 원하는 오브젝트 간 의존관계를 자바 코드로 직접 만들 수 있다.
```java

@Configuration
public class AppConfig {

     @Bean
     public FooService fooService() {
         return new FooService(fooRepository());
     }

     @Bean
     public FooRepository fooRepository() {
         return new JdbcFooRepository(RedisDataSource());
     }
 }

```
이때 @Configuration은 하나 이상의 @Bean 메서드를 선언하고, 런타임 시 해당 Bean에 대한 Bean 정의 및 서비스 요청을 생성하기 위해 Spring 컨테이너에서 처리될 수 있도록 한다.

그렇다면 @Component와 @Bean은 같이 사용할 수 없는 것일까? 그렇지 않다. 
@Component 애노테이션이 붙은 클래스 내부에 @Bean이 있어도, @Component는 컴포넌트 스캔의 대상이 되기 때문에 해당 Config 파일을 찾을 수 있다.  

```java
@Component
class AppConfig{
	@Bean
    public MemberService memberService() {
        return new MemberServiceImpl();
    }
}

```

```java
public class BeanTest {

    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);

    @Test
    void findApplicationBeanTest() {
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            BeanDefinition beanDefinition = ac.getBeanDefinition(beanDefinitionName);

            if (beanDefinition.getRole() == BeanDefinition.ROLE_APPLICATION) {
                Object bean = ac.getBean(beanDefinitionName);
                System.out.println("objectName = " + beanDefinitionName + ", object = " + bean);
            }
        }
    }
}
```

![](https://velog.velcdn.com/images/kmss6905/post/523ff162-ba76-44ed-8931-cd5f8669f9b5/image.png)
appConfig와 memberService라는 이름으로 두 객체가 빈으로 등록된 것을 볼 수 있다.

그렇다면 왜 굳이 @Configuration과 함께 @Bean을 사용할까?

관련해서 Spring에서는 다음과 같이 언급한다.

> You can use @Bean-annotated methods with any Spring @Component. However, they are most often used with @Configuration beans.
...
@Configuration classes let inter-bean dependencies be defined by calling other @Bean methods in the same class.

_@Configuration 클래스는 동일한 클래스 내의 다른 @Bean 메서드를 호출함으로써 빈 간의 의존성을 정의할 수 있다._

## @Component의 역할과 한계

@Component는 클래스 수준에서 빈을 정의하기 위해 사용하는 애노테이션이다.
해당 애노테이션이 붙은 클래스는 컴포넌트 스캔의 대상이 되어 빈으로 등록된다.

## @Configuration과 @Component의 차이 실험

실제로 @Configuration 대신 @Component로 바꾸고, 내부에 정의된 빈을 이용해 또 다른 빈을 만들려고 시도하면 아래와 같이 에러가 발생한다.
![](https://velog.velcdn.com/images/kmss6905/post/302fb9db-ea8d-4f5f-b69e-3e8e86fda03a/image.png)

## 정리: 언제 어떤 애노테이션을 써야 할까?

@Bean은 @Configuration과 함께 사용되어 메서드 레벨에서 직접 의존관계를 만든 후 빈으로 등록하고 싶을 때 사용한다.
또한 @Component는 클래스 수준에서 사용되는 애노테이션이기 때문에, 외부에 있는 클래스나 라이브러리를 빈으로 등록할 때는 어쩔 수 없이 @Bean을 사용한다.

클래스 단위에서 사용하고자 할 때는 @Component를 사용하여 빈으로 등록한다.

## 참고 자료
--- 
https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Bean.html
https://docs.spring.io/spring-framework/reference/core/beans/java/bean-annotation.html
https://jojoldu.tistory.com/27

