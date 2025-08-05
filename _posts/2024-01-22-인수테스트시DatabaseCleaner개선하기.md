
나는 인수테스트를 작성할 때 매번 테스트 실행 할 때마다 데이터베이스에 있는 모든 데이터를 초기화 시킨다. 
이를 위해 데이터베이스의 모든 테이블의 데이터를 초기화하는 역할을 담당하는 DatabaseCleaner.java 를 만들어 사용한다.

### 사용법
먼저 entitiyManger 의 기능을 이용하여 entity 의 이름을 얻고 이를 다시 가공하여 원하는 테이블의 이름으로 바꾼다. 이후 각 테이블이름을 가져다 초기화에 필요한 쿼리를 직접 데이터베이스에 날리는 식이다.
```java
entityManager.createNativeQuery("TRUNCATE TABLE " + tableName + ";").executeUpdate();
```

### 문제
이때 중요한 것은 엔티티의 이름을 이용해 실제 테이블 이름을 가져오는 것이다.
하지만 테이블의 이름과 엔티티의 이름이 전혀 다른 경우(가령 엔티티가 MentorJobType 가 매핑하고 있는 테이블의 이름이 JOB_TYPE 일 경우) 공통적으로 가공하여 실제 테이블 이름을 가져올 수 없다. 공통적으로 테이블 이름을 가져올 수 있어야한다.

### 기존코드
```java

package api.devmatching.accptance;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import java.util.List;
import java.util.Locale;
import java.util.stream.Collectors;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
public class DatabaseCleaner implements InitializingBean {

    @PersistenceContext
    private EntityManager entityManager;

    private List<String> tableNames;

    @Override
    public void afterPropertiesSet() {
      tableNames = entityNameToUpperCase();
    }

    private List<String> entityNameToUpperCase() {
        return entityManager.getMetamodel().getEntities()
            .stream().map(it -> it.getName().toUpperCase(Locale.ROOT))
            .collect(Collectors.toList());
    }

    @Transactional
    public void execute() {
        entityManager.flush();
        entityManager.createNativeQuery("SET foreign_key_checks = 0;").executeUpdate();
        tableNames.forEach(tableName -> executeQueryWithTable(tableName));
        entityManager.createNativeQuery("SET foreign_key_checks = 1;").executeUpdate();
    }

    private void executeQueryWithTable(String tableName) {
        entityManager.createNativeQuery("TRUNCATE TABLE " + tableName + ";").executeUpdate();
        entityManager.createNativeQuery("ALTER TABLE " + tableName + " AUTO_INCREMENT = 1;").executeUpdate();
    }
}

```

### 해결1

가공했을 때의 엔티티 이름과 테이블의 이름이 다른 경우에는 따로 제거하여, 실제 테이블이름을 리스트에 추가하는 방식이다. 이는 실제 테이블 이름과 엔티티 이름이 다른 경우를 미리 알고 있어야 하기 때문에 실수할 여지가 있다.

```java
@Override
  public void afterPropertiesSet() {
    tableNames = entityNameToUpperCase();
    
    // 테이블 이름과 일치 하지 않는 경우 따로 기존 tableNames 에서 제거한다.
    tableNames.stream().filter(it -> it.equals("MENTORJOBTYPE"))
        .toList()
        .forEach(it -> tableNames.remove(it));

    // 따로 일치하지 않는 테이블의 경우 직접 넣어준다.
    tableNames.add("JOB_TYPE");
  }
```



### 해결2

`@Table` 어노테이션이 붙은 엔티티를 대상으로 실제 테이블의 이름을 가져왔다. 따로 엔티티의 이름을 변환할 필요가 없이 @Table 어노테이션에 있는 name() 메서드 값을 가져올 수 있다.

```java
tableNames = entityManager.getMetamodel().getEntities().stream()
        .map(it -> it.getJavaType().getAnnotation(Table.class))
        .filter(Objects::nonNull)
        .map(Table::name)
        .collect(Collectors.toList());
```

### 유의
결론적으로 두 번째 방법을 사용했다. Entity 작성 시 `@Table` 어노테이션은 꼭 함께 작성하기 때문에 실수로 제외될 여지가 첫번 째 방법보다 적다고 판단했기 때문이다. 

물론, 이때 작성한 Entity 의 `@Table` 어노테이션이 없는 경우에는 테이블 이름 리스트에서 제외되기 때문에 반드시 `@Table` 어노테이션을 작성해야한다.


