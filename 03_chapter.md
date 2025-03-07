# Глава 3: Продвинутые возможности и Типы

## 1\. Продвинутый маппинг

### Встраиваемые объекты (Value Objects)

Не все данные заслуживают своей собственной таблицы. В DDD (Domain Driven Design) есть понятие **Value Object** (объект-значение) — например, `Address` или `Money`. У них нет своего ID, они часть сущности.

```java
@Embeddable
public class Address {
    private String city;
    private String street;
    private String zipCode;
    // ...
}

@Entity
public class User {
    @Id @GeneratedValue private Long id;
    
    @Embedded // Поля Address "распакуются" в таблицу users
    @AttributeOverrides({ // Можно переименовать колонки
        @AttributeOverride(name = "city", column = @Column(name = "billing_city"))
    })
    private Address billingAddress;
}
```

*Результат в БД:* Таблица `users` будет иметь колонки `id`, `billing_city`, `street`, `zipCode`. Таблицы `address` не существует.

### Маппинг Enums

**Критически важный момент.** По умолчанию Hibernate сохраняет Enum как `ORDINAL` (числовой индекс: 0, 1, 2...).
*Проблема:* Если вы вставите новое значение в середину Enum'а, все индексы съедут. Данные в базе превратятся в мусор.

**Решение:** Всегда используйте `STRING`.

```java
@Enumerated(EnumType.STRING) // Сохраняет "ACTIVE", "BANNED" текстом
private UserStatus status;
```

### Наследование (Inheritance)

Как положить иерархию классов Java в реляционную БД?

Есть три стратегии:

1.  **SINGLE\_TABLE (По умолчанию):**
      * Все поля всех наследников лежат в **одной гигантской таблице**.
      * Появляется колонка `DTYPE` (Discriminator), определяющая класс.
      * *Плюс:* Очень быстро (нет JOIN).
      * *Минус:* Нельзя поставить `NOT NULL` на поля наследников. Таблица пухнет.
2.  **JOINED (Нормализованная):**
      * Есть таблица родителя и отдельные таблицы для каждого наследника.
      * При выборке Hibernate делает `JOIN`.
      * *Плюс:* Чистая схема БД, работают Constraints.
      * *Минус:* Медленно при выборке (много JOIN-ов) и вставке (несколько INSERT).
3.  **TABLE\_PER\_CLASS:**
      * Таблица родителя не создается (обычно он абстрактный). У каждого наследника своя таблица со всеми полями (своими + родительскими).
      * *Минус:* Полиморфные запросы (найти всех `Animal`) очень медленные, так как используется `UNION ALL` по всем таблицам.

-----

## 2\. Продвинутое построение запросов

Когда `findByEmail` уже не хватает (например, фильтр в интернет-магазине с 20 параметрами, половина из которых может быть null), нам нужны динамические запросы.

### Criteria API

Это стандартный способ строить запросы программно, без склейки строк (String Concatenation), что защищает от SQL Injection. Однако чистый JPA Criteria API очень многословен и сложен для чтения.

### Spring Data Specifications

Это элегантная обертка над Criteria API. Вы создаете маленькие кусочки логики ("Спецификации") и комбинируете их.

```java
// Спецификация: "Цена больше чем X"
public static Specification<Product> hasPriceGreaterThan(BigDecimal price) {
    return (root, query, criteriaBuilder) -> 
        criteriaBuilder.greaterThan(root.get("price"), price);
}

// Спецификация: "В категории Y"
public static Specification<Product> isInCategory(String category) {
    return (root, query, criteriaBuilder) -> 
        criteriaBuilder.equal(root.get("category"), category);
}

// Использование в сервисе:
repository.findAll(
    Specification.where(hasPriceGreaterThan(100))
    .and(isInCategory("Electronics"))
);
```

Для этого репозиторий должен наследовать `JpaSpecificationExecutor`.

### Projections (Проекции)

Частая ошибка: загружать тяжелую сущность `User` со всеми связями, чтобы просто отобразить "Имя" и "Фамилию" в списке. Это убивает память.

Spring Data позволяет использовать **интерфейсные проекции**:

```java
// Интерфейс с геттерами только нужных полей
public interface UserSummary {
    String getFirstName();
    String getLastName();
    // Spring даже может вычислить значение (Open Projection), но это медленнее
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName(); 
}

// В репозитории
List<UserSummary> findByActiveTrue(); 
```

Spring сгенерирует SQL запрос, который выберет **только** указанные поля, а не `SELECT *`.

-----