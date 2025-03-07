# Глава 6. Тонкая настройка данных: Конвертеры и Soft Delete

В этой главе мы научимся двум вещам:

1.  Хранить данные в одном виде (например, зашифрованном), а в Java работать с ними в другом.
2.  Удалять данные так, чтобы они оставались в базе (Soft Delete).

-----

## 1\. Attribute Converters (`@Converter`)

Иногда типы данных в базе и в Java не совпадают.

  * **Пример 1:** В базе поле `gender` — это `CHAR(1)` ('M', 'F'), а в Java это `Enum Gender`. (Решается через `@Enumerated`, но конвертер гибче).
  * **Пример 2 (Киллер-фича):** Шифрование. В базе мы хотим хранить `AES:83h1289...`, а в Java видеть `1234-5678-9000`.

### Как это работает

Hibernate вызывает ваш код **перед** вставкой в БД (`convertToDatabaseColumn`) и **сразу после** чтения из БД (`convertToEntityAttribute`).

### Реализация (Шифрование данных)

Сначала создаем сам конвертер:

```java
@Converter
// <Тип_в_Java, Тип_в_БД>
public class CryptoConverter implements AttributeConverter<String, String> {

    @Override
    public String convertToDatabaseColumn(String sensitiveData) {
        if (sensitiveData == null) return null;
        return encrypt(sensitiveData); // Ваш метод шифрования (AES/RSA)
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        return decrypt(dbData); // Ваш метод расшифровки
    }
}
```

Применяем в сущности:

```java
@Entity
public class User {
    @Id @GeneratedValue private Long id;

    @Convert(converter = CryptoConverter.class)
    private String creditCardNumber; 
    // В Java это "4444...", в БД лежит "x8s7f6..."
}
```

### 🪨 Подводные камни

1.  **Поиск по полю:** Вы **не сможете** эффективно искать по этому полю (`findByCreditCardNumber`). База видит абракадабру. Поиск придется делать в памяти или использовать детерминированное шифрование (что менее безопасно).
2.  **`autoApply = true`:** Можно написать `@Converter(autoApply = true)`. Тогда конвертер применится **ко всем** полям типа `String` во всем проекте. Это опасно.
3.  **Частичные данные:** Если вы используете `JPQL` конструкцию `select u.creditCardNumber from User u`, конвертер сработает. Но если вы используете Native SQL (`select credit_card from users`), вы получите зашифрованную строку.

-----

## 2\. Soft Delete (Мягкое удаление)

В корпоративных системах удалять данные физически (`DELETE`) часто запрещено. Нужна история, аудит или возможность восстановления.

**Задача:**

1.  При вызове `repository.delete(id)` делать `UPDATE users SET deleted = true WHERE id = ?`.
2.  При вызове `findAll()` не показывать удаленные записи.

### Реализация (Современный подход Hibernate 6.3+)

Раньше использовали аннотацию `@Where`, но она объявлена **Deprecated**. Теперь используем `@SQLDelete` и `@SQLRestriction`.

```java
@Entity
@Table(name = "users")
// 1. Перехватываем команду DELETE
@SQLDelete(sql = "UPDATE users SET deleted = true WHERE id = ?")
// 2. Добавляем фильтр ко ВСЕМ запросам чтения (SELECT, JOIN и т.д.)
@SQLRestriction("deleted = false") 
public class User {

    @Id @GeneratedValue private Long id;

    private String username;

    // Флаг удаления
    @Column(nullable = false)
    private boolean deleted = false;
}
```

### Как это выглядит в коде

```java
userRepository.deleteById(1L); 
// Hibernate выполнит: UPDATE users SET deleted = true WHERE id = 1

List<User> users = userRepository.findAll();
// Hibernate выполнит: SELECT * FROM users WHERE deleted = false
```

### 🪨 Подводные камни (Expert Level)

Это тема, на которой часто "ломают копья".

1.  **Уникальные индексы (Unique Constraints):**

      * Представьте, что у `username` стоит `UNIQUE`.
      * Вы удаляете пользователя "admin" (он становится `deleted=true`).
      * Вы пытаетесь создать нового пользователя "admin".
      * **Ошибка БД\!** Физически старая запись "admin" всё еще там.
      * *Решение:* Использовать "Partial Index" в PostgreSQL:
        `CREATE UNIQUE INDEX idx_username ON users (username) WHERE deleted = false;`

2.  **Native SQL:**

      * Если вы напишете `@Query(value = "SELECT * FROM users", nativeQuery = true)`, Hibernate **не добавит** условие `deleted = false`. Нативные запросы игнорируют аннотации сущности. Вы получите удаленные записи.

3.  **Каскадное удаление:**

      * Если у `User` есть `List<Order>`, и у связи стоит `CascadeType.REMOVE`, то при удалении User Hibernate попытается удалить и Order.
      * Если у Order тоже стоит `@SQLDelete`, всё пройдет хорошо (будет цепочка Update-ов).
      * Если нет — Hibernate сделает физический `DELETE` для ордеров.

-----

## 3\. Entity Listeners (Обратные вызовы)

JPA позволяет вынести логику из методов `save/update` во внешние классы.

**Пример:** Мы хотим логировать изменения или отправлять уведомления, но не хотим засорять код Сервиса или Сущности.

```java
public class UserAuditListener {

    @PrePersist // Перед INSERT
    public void beforeCreate(User user) {
        System.out.println("Создаем юзера: " + user.getUsername());
    }

    @PostPersist // После успешного INSERT
    public void afterCreate(User user) {
        // Пример: Отправить событие в Kafka
        // EmailService.sendWelcomeEmail(user.getEmail()); 
    }
}

// Подключаем к сущности
@Entity
@EntityListeners(UserAuditListener.class)
public class User { ... }
```

### 🪨 Важное ограничение

Внутри `EntityListener` **нельзя** использовать репозитории или EntityManager для изменения БД. Вы находитесь внутри процесса "flush". Попытка сохранить другую сущность здесь может привести к непредсказуемому поведению или рекурсии.

-----