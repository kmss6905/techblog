---
layout: post
title: "Java의 final은 어디에 사용할 수 있을까?"
description: "자바의 final 키워드는 클래스, 메서드, 변수에 사용하여 재할당을 막을 수 있습니다. 이 글에서는 final의 다양한 사용법과 주의할 점을 예제와 함께 자세히 알아봅니다."
categories:
- "Java"
tags:
- "java"
- "final"
- "불변성"
- "immutable"
date: "2024-01-07 00:00:00 +0900"
toc: true
---

# Java `final` 키워드 완벽 정복

> 예제에서 사용한 모든 코드는 [Github Repository](https://github.com/kmss6905/blog/tree/main/_20240107) 에 있습니다.

`final` 키워드는 주로 변경 불가능한 전역 변수를 선언하거나 생성자의 파라미터로 받을 때 사용됩니다. 하지만 `final`은 이보다 더 다양한 곳에서 활용될 수 있습니다. 이 글에서는 `final` 키워드의 다양한 사용법을 자세히 알아보고, 예제를 통해 실제 활용 방법을 살펴봅니다.

## `final` 키워드 사용법

`final` 키워드는 크게 **클래스**, **메서드**, **변수** 세 가지 대상에 사용할 수 있습니다.

### 1. `final` 클래스: 상속을 막는 마지막 보루

클래스에 `final` 키워드를 사용하면 해당 클래스는 더 이상 상속할 수 없습니다. 이는 클래스의 확장을 원하지 않을 때 유용하게 사용됩니다.

```java
public final class Cat {
    private int weight;
    // standard getter and setter
}

public class BlackCat extends Cat { // 컴파일 에러!
}
```

![final 클래스 상속 불가](https://velog.velcdn.com/images/kmss6905/post/f64a3029-421e-4add-b1b8-1427671ce17e/image.png)

**주의할 점:** `final` 클래스가 불변(immutable) 객체를 의미하는 것은 아닙니다. 클래스 내부의 멤버 변수는 얼마든지 변경될 수 있습니다.

```java
class ClassFinalMainTest {

    @Test
    @DisplayName("final 클래스의 맴버변수는 바꿀 수 있다.")
    void mainTest() {
        Money money = new Money();
        money.setValue(100);

        assertEquals(100, money.getValue());
        assertDoesNotThrow(() -> money.setValue(200)); // 예외발생하지 않음.
        assertEquals(200, money.getValue());
    }

    final class Money {
        private int value;

        public void setValue(int value) {
            this.value = value;
        }

        public int getValue() {
            return value;
        }
    }
}
```

![final 클래스 테스트 결과](https://velog.velcdn.com/images/kmss6905/post/8e6d87d9-fa5f-4015-9a64-751f5f184dc7/image.png)

> **Tip:** IntelliJ IDEA에서는 `final` 클래스에 "압정" 아이콘을 표시하여 상속할 수 없음을 시각적으로 알려줍니다.
>
> ![IntelliJ final 클래스 표시](https://velog.velcdn.com/images/kmss6905/post/83e18ef2-ee70-4c65-af5a-07c36acbf096/image.png)

### 2. `final` 메서드: 오버라이딩 금지

메서드에 `final` 키워드를 사용하면 해당 메서드는 하위 클래스에서 오버라이딩(재정의)할 수 없습니다.

```java
// 부모 클래스
public class Cat {
    public void meow() {
        System.out.println("누구나 야옹~");
    }

    final public void finalMeow() {
        System.out.println("나만 야옹~");
    }

    private void privateMeow() {
        System.out.println("내부 야옹~");
    }
}

// 자식 클래스
public class WhiteCat extends Cat {
    @Override
    public void meow() {
        System.out.println("흰 고양이 야옹");
    }

    // finalMeow() 메서드를 오버라이딩 하려고 하면 컴파일 에러 발생!
    // @Override
    // public void finalMeow() { ... }
}
```

만약 자식 클래스에서 부모의 `final` 메서드를 재정의하려고 하면 `'finalMeow()' cannot override 'finalMeow()' in 'Cat'; overridden method is final` 와 같은 컴파일 에러가 발생합니다.

![final 메서드 오버라이딩 불가](https://velog.velcdn.com/images/kmss6905/post/b3686362-1048-4d20-a745-d13bd812a7a7/image.png)

### 3. `final` 변수: 재할당은 이제 그만

변수에 `final` 키워드를 사용하면 **한 번만 값을 할당**할 수 있으며, 이후에는 값을 변경할 수 없습니다.

#### 3.1. 원시 변수 (Primitive Variables)

`final`로 선언된 원시 변수는 초기화 이후 다른 값을 할당할 수 없습니다.

```java
final int i = 1;
i = 2; // 컴파일 에러!
```

![final 원시 변수 재할당 불가](https://velog.velcdn.com/images/kmss6905/post/2317ff3d-d4a3-4ade-8361-fdd4b990c068/image.png)

#### 3.2. 참조 변수 (Reference Variables)

`final`로 선언된 참조 변수는 다른 객체를 참조하도록 변경할 수 없습니다.

```java
final User user = new User("jimin");
user = new User("junguk"); // 컴파일 에러!
```

![final 참조 변수 재할당 불가](https://velog.velcdn.com/images/kmss6905/post/19979a37-0fa3-470d-87b9-57d5d877ebfd/image.png)

**중요:** 참조 변수 자체가 다른 객체를 가리키도록 변경할 수는 없지만, 참조하고 있는 **객체의 내부 상태(멤버 변수)는 변경 가능**합니다.

```java
final class XXXclass {
    private int a = 5;
}

final XXXClass xxxClass = new XXXClass();
xxxClass.a = 10; // 수정 가능!
```

#### 3.3. 필드 (Fields)

`final`은 클래스의 필드에도 사용할 수 있습니다. 주로 상수(constant)를 정의하거나, 생성자에서 초기화되는 멤버 변수에 사용됩니다.

**상수 필드:**

```java
class Point {
    private static final double GLOBAL_POINT = 10.0;

    public void changePointToTenDotOne() {
        // GLOBAL_POINT = 10.1; // 컴파일 에러!
    }
}
```

![final 상수 필드 재할당 불가](https://velog.velcdn.com/images/kmss6905/post/776c5300-0757-485a-81b6-0ab61176fdc8/image.png)

**생성자 멤버 변수:**

`final`로 선언된 멤버 변수는 생성자 내에서 반드시 초기화되어야 합니다. 이는 객체의 불변성을 보장하고, `NullPointerException`과 같은 실수를 방지하는 데 도움을 줍니다.

```java
public class OrderService {
    private final ProductRepository productRepository; // final 키워드 추가

    public OrderService(ProductRepository productRepository) {
        this.productRepository = productRepository; // 생성자에서 반드시 초기화
    }

    public void order(int id) {
        Product product = productRepository.findId(id);
        // etc
    }
}
```

만약 `final` 멤버 변수를 생성자에서 초기화하지 않으면, 컴파일 에러가 발생하여 실수를 미리 방지할 수 있습니다.

![final 멤버 변수 초기화 강제](https://velog.velcdn.com/images/kmss6905/post/d9b47b0f-1888-4624-869b-4b3c51f67244/image.png)

#### 3.4. 메서드 인자 (Argument Variables)

메서드의 인자에 `final` 키워드를 사용하면, 메서드 내부에서 해당 인자의 값을 변경할 수 없습니다.

```java
public int plus(final int a, final int b) {
    // a += b; // 컴파일 에러!
    int c = a + b;
    return c;
}
```

## 요약

Java의 `final` 키워드는 **클래스, 메서드, 변수**에 사용하여 **재할당을 방지**하는 역할을 합니다.

- **`final` 클래스:** 상속 불가
- **`final` 메서드:** 오버라이딩 불가
- **`final` 변수:** 재할당 불가

특히 생성자의 멤버 변수에 `final`을 사용하면, 객체 생성 시점에 해당 변수가 반드시 초기화되도록 강제하여 코드의 안정성을 높일 수 있습니다. `final` 키워드를 적재적소에 활용하여 더 견고하고 예측 가능한 코드를 작성해 보세요.

---

### 참조

- [Baeldung - The final Keyword in Java](https://www.baeldung.com/java-final)