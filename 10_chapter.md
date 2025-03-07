# Глава 10. Тестирование

## 1\. Зависимости (pom.xml)

Вам понадобятся эти библиотеки (версии Spring Boot подтянет сам, если используете `spring-boot-starter-parent`):

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

-----

## 2\. Базовый класс конфигурации

Чтобы не поднимать контейнер для каждого тест-класса заново (это долго), мы создадим абстрактный класс. Контейнер запустится один раз для всех тестов.

```java
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
public abstract class AbstractIntegrationTest {

    // Поднимаем Docker-контейнер с Postgres 15
    private static final PostgreSQLContainer<?> POSTGRES = 
        new PostgreSQLContainer<>("postgres:15-alpine")
            .withReuse(true); // Оптимизация для локальной разработки

    static {
        POSTGRES.start();
    }

    // Переписываем настройки application.properties на лету, 
    // чтобы Spring подключился к этому контейнеру
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        
        // Важно: отключаем ddl-auto=update, лучше использовать flyway/liquibase. 
        // Но для теста сущностей допустимо create-drop
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "create-drop");
    }
}
```

-----

## 3\. Сам тест (UserRepositoryTest)

Здесь мы проверяем всё: сохранение, каскады, генерацию ID, аудит и удаление сирот.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.context.annotation.Import;

import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest // Поднимает только JPA контекст (быстро)
// Отключаем попытку Spring заменить базу на H2. Мы хотим наш Docker!
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest extends AbstractIntegrationTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager; // Позволяет работать с контекстом напрямую

    @Test
    void shouldGenerateIdAndAuditFields() {
        // Given
        User user = new User();
        user.setUsername("test_user");

        // When
        User savedUser = userRepository.save(user);
        
        // Сбрасываем кэш (flush & clear), чтобы заставить Hibernate сходить в БД 
        // при следующем чтении. Иначе мы проверим только кэш.
        entityManager.flush();
        entityManager.clear();

        // Then
        User foundUser = userRepository.findById(savedUser.getId()).orElseThrow();
        
        assertThat(foundUser.getId()).isNotNull(); // ID сгенерирован (Sequence)
        assertThat(foundUser.getCreatedAt()).isNotNull(); // @CreationTimestamp сработал
        assertThat(foundUser.getStatus()).isEqualTo(UserStatus.ACTIVE); // Default value
    }

    @Test
    void shouldPersistOrdersCascade() {
        // Given
        User user = new User();
        user.setUsername("buyer");

        Order order1 = new Order();
        order1.setDescription("Laptop");
        
        Order order2 = new Order();
        order2.setDescription("Mouse");

        // Используем наш helper-метод!
        user.addOrder(order1);
        user.addOrder(order2);

        // When
        userRepository.save(user); // Сохраняем ТОЛЬКО юзера
        
        entityManager.flush();
        entityManager.clear();

        // Then
        User foundUser = userRepository.findById(user.getId()).get();
        
        assertThat(foundUser.getOrders()).hasSize(2);
        assertThat(foundUser.getOrders().get(0).getUser()).isNotNull(); // Связь установлена
    }

    @Test
    void shouldRemoveOrphan() {
        // 1. Создаем юзера с ордером
        User user = new User();
        user.setUsername("deleter");
        Order order = new Order();
        order.setDescription("To delete");
        user.addOrder(order);
        
        user = userRepository.save(user);
        entityManager.flush();
        entityManager.clear();

        // 2. Загружаем и удаляем ордер из коллекции
        User loadedUser = userRepository.findById(user.getId()).get();
        Order loadedOrder = loadedUser.getOrders().get(0);
        
        // Используем helper для разрыва связи
        loadedUser.removeOrder(loadedOrder);
        
        // Сохраняем изменения (User)
        userRepository.save(loadedUser);
        entityManager.flush();
        entityManager.clear();

        // 3. Проверяем, что ордер исчез из БД
        Order deletedOrder = entityManager.find(Order.class, loadedOrder.getId());
        assertThat(deletedOrder).isNull(); // orphanRemoval = true сработал!
    }
}
```

## На что обратить внимание в этом коде:

1.  `entityManager.flush()` и `clear()`: Это **критически важно** в тестах `@DataJpaTest`. По умолчанию транзакция длится весь тест. Если вы сохраните объект и тут же попытаетесь его найти (`findById`), Hibernate вернет вам объект **из памяти** (L1 Cache), даже не делая запрос в БД. Вызовы `flush` (отправить изменения в БД) и `clear` (очистить память) гарантируют, что следующий запрос реально пойдет в базу данных Postgre и проверит, как всё сохранилось на самом деле.
2.  `@AutoConfigureTestDatabase(replace = NONE)`: Без этой строки Spring проигнорирует ваши настройки Testcontainers и попытается запустить H2.
3.  **Тестирование сирот (Orphan Removal):** Последний тест доказывает, что удаление объекта из Java-списка `orders` реально приводит к `DELETE FROM orders` в базе.

Этот тест — ваш "страховой полис". Если кто-то случайно уберет `orphanRemoval = true` или сломает `hashCode`, эти тесты упадут.