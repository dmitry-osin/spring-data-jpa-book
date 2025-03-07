# Глава 12. Идеальная Entity

## 1\. Базовый класс (Auditing & MappedSuperclass)

Выносим технические поля (кто создал, когда обновил), чтобы не мусорить в бизнес-классах.

```java
@MappedSuperclass // Поля этого класса попадут в таблицы наследников
@EntityListeners(AuditingEntityListener.class) // Включает магию Spring Data Auditing
@Getter
@Setter
public abstract class BaseEntity {

    @CreationTimestamp // Hibernate сам поставит время при INSERT
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp // Hibernate обновит время при каждом UPDATE
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

-----

## 2\. Идеальная Сущность (User)

Обратите внимание на комментарии — там вся соль.

```java
import lombok.*;
import org.hibernate.annotations.BatchSize;
import org.hibernate.proxy.Hibernate;

import jakarta.persistence.*;
import java.util.*;

@Entity
@Table(name = "users")
// 1. LOMBOK: Не используем @Data!
@Getter
@Setter
@NoArgsConstructor // Обязателен для Hibernate
@ToString // Безопасный toString настроим ниже
public class User extends BaseEntity {

    @Id
    // 2. SEQUENCE: Позволяет использовать Batch Insert (вставку пачками)
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
    @SequenceGenerator(name = "user_seq", sequenceName = "users_id_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    // 3. ENUM: Только STRING!
    @Enumerated(EnumType.STRING)
    private UserStatus status = UserStatus.ACTIVE;

    // 4. ОТНОШЕНИЯ: Inverse side (Обратная сторона)
    @OneToMany(
        mappedBy = "user", // Ссылаемся на поле 'user' в классе Order
        cascade = CascadeType.ALL, // Сохраняем User -> сохраняются Orders
        orphanRemoval = true // Удаляем Order из списка -> DELETE из БД
    )
    // 5. FETCHING: Решаем N+1 для коллекций
    @BatchSize(size = 20) 
    // 6. LOMBOK: Исключаем ленивые поля из toString, иначе будет SELECT или ошибка
    @ToString.Exclude 
    private List<Order> orders = new ArrayList<>();

    // 7. HELPER METHODS: Синхронизация двунаправленной связи
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this); // Важно!
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setUser(null);
    }

    // 8. EQUALS & HASHCODE: Самая сложная часть (Proxy-safe)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        // Важно: проверяем через Hibernate.getClass, так как 'o' может быть Proxy
        if (o == null || Hibernate.getClass(this) != Hibernate.getClass(o)) return false;
        User user = (User) o;
        // Сравниваем только ID. Если ID null (transient) -> объекты разные
        return id != null && Objects.equals(id, user.id);
    }

    @Override
    public int hashCode() {
        // Константа! Это предотвращает изменение хэш-кода при первом сохранении (когда появляется ID).
        // Да, это превращает HashMap в Linked list в худшем случае, но это безопасно.
        return getClass().hashCode();
    }
}
```

-----

## 3\. Вторая сторона (Order)

```java
@Entity
@Table(name = "orders")
@Getter
@Setter
@NoArgsConstructor
public class Order extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    private String description;

    // 9. LAZY: Обязательно меняем EAGER на LAZY
    @ManyToOne(fetch = FetchType.LAZY) 
    @JoinColumn(name = "user_id") // Владеющая сторона (тут FK)
    @ToString.Exclude // Исключаем, чтобы не зациклить toString (User -> Order -> User...)
    private User user;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || Hibernate.getClass(this) != Hibernate.getClass(o)) return false;
        Order order = (Order) o;
        return id != null && Objects.equals(id, order.id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

-----

## Почему именно так? (Разбор ключевых решений)

### Почему `hashCode` возвращает константу?

Это неочевидный момент.

1.  Вы создали `User u = new User()`. У него `id = null`.
2.  Вы положили его в `HashSet`. Хэш посчитался (например, от случайного адреса памяти или 0).
3.  Вы сохранили `repository.save(u)`. Ему присвоился `id = 100`.
4.  Если бы `hashCode` зависел от `id`, он бы **изменился**.
5.  Теперь `HashSet` не может найти этот объект, потому что он лежит в "старой" корзине (bucket), а ищется в новой. **Memory Leak гарантирован.**
    Возврат `getClass().hashCode()` решает это, делая хэш стабильным в течение всей жизни объекта.

### Почему `Hibernate.getClass(this)`?

Если вы загружаете `User` лениво (через `getReferenceById` или как связь), Hibernate подсовывает вам не класс `User`, а класс `User$HibernateProxy$Zw3s...`.
Обычный `this.getClass()` вернет разные классы для оригинала и прокси, и `equals` вернет `false`, хотя это одна и та же запись в БД.

### Почему `@BatchSize`?

Когда вы будете бежать циклом по `users` и вызывать `getOrders()`, без этой аннотации Hibernate делал бы по одному запросу на каждого юзера. С ней он подгрузит ордера сразу для 20 (или 50) юзеров за один раз. Это дешевое решение N+1 без написания сложных запросов.

### Почему `allocationSize = 50`?

Стандартное значение часто 50, но иногда люди ставят 1.
Если `allocationSize = 1`, Hibernate будет ходить в базу за новым ID **каждый раз** при создании объекта.
Если `allocationSize = 50`, он один раз сходит в базу, получит диапазон (например, 1000-1050) и следующие 50 объектов сохранит в памяти очень быстро, а потом отправит их в БД одним пакетом (Batch).

-----