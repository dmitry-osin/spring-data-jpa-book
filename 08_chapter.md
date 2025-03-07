# Глава 8. Продвинутые запросы и Аналитика

## 1\. Проблема чистого JPA и Native Queries

Стандартный JPQL хорош для загрузки объектов, но плох для аналитики.
Чего нет в старых версиях (или неудобно в новых):

  * **Window Functions** (`ROW_NUMBER() OVER...`, `RANK()`, `PARTITION BY`).
  * **CTEs** (`WITH RECURSIVE` — для деревьев и иерархий).
  * **Slozhnyye JOIN-ы** (например, `LATERAL JOIN`).

### Решение 1: Native Query (Лобовая атака)

Мы пишем чистый SQL внутри аннотации.

```java
@Query(value = """
    SELECT u.username, sum(o.total) as total_sum,
           RANK() OVER (ORDER BY sum(o.total) DESC) as rank
    FROM users u
    JOIN orders o ON u.id = o.user_id
    GROUP BY u.username
    """, nativeQuery = true)
List<Object[]> findUserRanks(); // Возвращает массив объектов :(
```

### 🪨 Подводные камни

1.  **Маппинг (ResultSetMapping):** Возвращать `List<Object[]>` — это ад. Вам придется вручную кастить `(String) obj[0]`, `(BigDecimal) obj[1]`.
      * *Решение:* Использовать **Interface Projections** (Spring Data сам смапит колонки по именам геттеров) или `@SqlResultSetMapping` (очень многословно).
2.  **Зависимость от БД:** Если вы напишете запрос на диалекте PostgreSQL (используя `jsonb_agg`), вы не сможете запустить это на H2 или MySQL.
3.  **Отсутствие проверки типов:** Если вы переименуете поле в Entity, этот запрос сломается только в Runtime (когда вы его запустите).

-----

## 2\. Blaze-Persistence (Секретное оружие)

Это библиотека, которая работает *поверх* Hibernate. Она позволяет писать JPQL-подобный код, но с поддержкой **всех** фич современного SQL (CTE, Window Functions, Values clause, Union).

Это выбор экспертов для сложных проектов.

### Пример: CTE (Иерархия категорий)

Допустим, у нас есть дерево категорий (Electronics -\> Laptops -\> Gaming), и нам нужно выбрать всё поддерево одним запросом.

```java
// CriteriaBuilderFactory внедряется Blaze-Persistence
CriteriaBuilder<Category> cb = cbf.create(em, Category.class)
    .withRecursive(CategoryCTE.class) // Объявляем CTE
        .from(Category.class, "c")
        .bind("id").select("c.id")
        .bind("parent").select("c.parent")
        .where("c.id").eq(rootCategoryId) // Стартовая точка
    .unionAll() // Рекурсивная часть
        .from(Category.class, "c")
        .join(CategoryCTE.class, "cte").on("cte.id").eq("c.parent.id") // Join с собой
        .bind("id").select("c.id")
        .bind("parent").select("c.parent")
    .end()
    .select("cte")
    .from(CategoryCTE.class, "cte"); // Финальная выборка

List<Category> tree = cb.getResultList();
```

**Плюсы:**

  * **Type Safe:** Ошибки видны на этапе компиляции.
  * **Entity View:** Возвращает не массив байтов, а типизированные объекты или DTO.
  * **Performance:** Генерирует оптимизированный SQL.

-----

## 3\. Продвинутые Проекции (Projections)

Мы уже говорили, что тянуть сущности целиком — дорого. Spring Data дает три уровня проекций.

### Уровень 1: Интерфейсы (Было выше)

Самый простой вариант. Spring генерирует прокси на лету.

### Уровень 2: Class-based (DTO) — Самый быстрый

Вы пишете обычный Java класс (POJO) с конструктором.

```java
// DTO (Lombok @Value делает класс неизменяемым и добавляет конструктор)
@Value
public class UserStatDto {
    String username;
    Long orderCount;
}

// Repository
// Внимание: нужно указывать полный путь к классу!
@Query("SELECT new com.example.dto.UserStatDto(u.username, count(o)) " +
       "FROM User u JOIN u.orders o GROUP BY u.username")
List<UserStatDto> findUserStats();
```

  * *Плюс:* Никаких прокси. Hibernate сразу вызывает конструктор (`new ...`). Максимальная скорость.
  * *Минус:* Длинные имена пакетов в запросе (`com.example...`).

### Уровень 3: Динамические проекции

Когда один метод репозитория может возвращать разные данные в зависимости от вызова.

```java
// Repository
<T> List<T> findByUsername(String username, Class<T> type);

// Service
// Хочу полную сущность
List<User> users = repo.findByUsername("alice", User.class);
// Хочу только краткую сводку
List<UserSummary> summaries = repo.findByUsername("alice", UserSummary.class);
// Хочу DTO
List<UserDto> dtos = repo.findByUsername("alice", UserDto.class);
```

### 🪨 Подводный камень (Dynamic Projections)

Динамические проекции отлично работают с Derived Queries (из имени метода). Но если вы используете `@Query`, вам придется использовать SpEL (Spring Expression Language), что усложняет код, или писать отдельные методы.

-----

## 4\. Hibernate 6: Дедупликация

Маленькая, но важная деталь при переходе на Spring Boot 3.

**В Hibernate 5:**
Если вы делали `LEFT JOIN FETCH` для коллекции (User + Orders), результат задваивался (User возвращался столько раз, сколько у него ордеров). Нужно было писать `SELECT DISTINCT u ...`.
При этом `DISTINCT` часто реально передавался в SQL, заставляя базу делать тяжелую сортировку.

**В Hibernate 6:**
Hibernate делает дедупликацию **в памяти** автоматически.

  * Вам больше не нужно писать `DISTINCT` в JPQL для предотвращения дублей сущностей.
  * Если вы все-таки напишете `DISTINCT`, Hibernate 6 попытается понять: нужен он в SQL или только для логики.

-----

**Итог главы:**

  * Если нужно «просто и быстро» что-то сложное — берите **Native Query** + **Interface Projection**.
  * Если строите сложную аналитическую систему со слоями — учите **Blaze-Persistence**.
  * Всегда используйте **DTO проекции** для списков (read-only операций).