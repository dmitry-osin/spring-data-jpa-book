# Глава 2: Отношения и Базовый Spring Data

## 1\. Маппинг отношений (Associations)

Главная сложность здесь — понять разницу между объектной моделью и реляционной.

  * **В БД**: Отношения строятся через внешние ключи (Foreign Key). Ключ всегда находится *в одной* таблице (кроме ManyToMany).
  * **В Java**: Объекты ссылаются друг на друга. Ссылки могут быть в обе стороны.

### Владеющая сторона (Owning Side) vs Обратная сторона (Inverse Side)

Это концепция, на которой ломается 90% новичков.

  * **Owning Side (Владеющая сторона):** Это сторона, у которой в таблице БД физически находится колонка `FOREIGN KEY`. Hibernate смотрит **только** на эту сторону, чтобы понять, что нужно сохранить в базу.
  * **Inverse Side (Обратная сторона):** Это сторона, которая просто ссылается на другую, используя `mappedBy`. Она нужна только для удобства программиста (чтобы получить список постов у юзера), но Hibernate игнорирует изменения в этой коллекции при сохранении связей, если не обновлена владеющая сторона.

### Пример: One-To-Many (Один ко Многим)

Классика: У одного `User` есть много `Post`.

**Владеющая сторона (Post):**

```java
@Entity
public class Post {
    @Id 
    @GeneratedValue 
    private Long id;

    private String title;

    @ManyToOne(fetch = FetchType.LAZY) // ВСЕГДА LAZY (подробнее в ур.4)
    @JoinColumn(name = "user_id") // Имя колонки FK в таблице posts
    private User user; // <-- Это поле управляет связью!
    
    // геттеры, сеттеры
}
```

**Обратная сторона (User):**

```java
@Entity
public class User {
    @Id 
    @GeneratedValue 
    private Long id;

    // mappedBy = "user" означает: "Иди в класс Post, найди поле 'user'. 
    // Вот ОНО управляет связью. Я здесь просто для чтения."
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Post> posts = new ArrayList<>();
    
    // ВАЖНО: Хелпер-метод для синхронизации
    public void addPost(Post post) {
        posts.add(post);
        post.setUser(this); // <-- Обязательно связываем владеющую сторону!
    }
}
```

> **Золотое правило:** При создании двунаправленной связи вы обязаны обновлять обе стороны\!
> `user.getPosts().add(post)` — обновит только кэш в Java.
> `post.setUser(user)` — создаст запись во внешнем ключе в БД.

### Каскадные операции (Cascade) и Orphan Removal

  * **CascadeType.PERSIST**: Сохраняю юзера -\> автоматически сохраняются его новые посты.
  * **CascadeType.ALL**: Включает в себя всё (сохранение, обновление, удаление).
  * **orphanRemoval = true**: Если я удалю пост из списка `user.getPosts().remove(post)`, Hibernate отправит `DELETE` запрос в БД. Без этой настройки связь просто разорвется (FK станет NULL), но запись останется (если нет constraint).

-----

## 2\. Знакомство со Spring Data JPA

Spring Data JPA — это абстракция над Hibernate. Она позволяет не писать реализацию DAO (Data Access Object) вручную. Вы объявляете интерфейс, Spring генерирует реализацию на лету (через Proxy).

### Иерархия интерфейсов

1.  **Repository**: Маркерный интерфейс.
2.  **CrudRepository**: Базовые методы (`save`, `findById`, `delete`, `count`).
3.  **JpaRepository**: Расширяет PagingAndSortingRepository. Добавляет JPA-специфичные методы (flush, saveAllAndFlush). Обычно наследуются именно от него.

<!-- end list -->

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Реализация не нужна! Spring сам поймет, что делать.
}
```

### Магия Derived Queries (Запросы из имени метода)

Spring парсит имя метода и превращает его в SQL.

```java
// SELECT * FROM users WHERE email = ?
Optional<User> findByEmail(String email);

// SELECT * FROM users WHERE active = true AND age > ?
List<User> findByActiveTrueAndAgeGreaterThan(int age);

// SELECT count(*) FROM users WHERE name LIKE ?
long countByNameStartingWith(String prefix);
```

*Плюс:* Быстро, читаемо для простых запросов.
*Минус:* Имя метода может стать монструозным (`findByNameAndAgeAndActiveTrueAnd...`).

### @Query (Когда имя метода слишком длинное)

Позволяет писать запросы на **JPQL** (Java Persistence Query Language). В JPQL мы оперируем *именами классов и полей*, а не таблиц и колонок.

```java
@Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
User findActiveUserByEmail(@Param("email") String email);
```

### Пагинация и Сортировка

Никогда не возвращайте `List<User>`, если в таблице миллион записей. Используйте `Pageable`.

```java
// В репозитории
Page<User> findAll(Pageable pageable);

// В сервисе
// Страница 0 (первая), размер 10, сортировка по ID убыванию
Pageable pageRequest = PageRequest.of(0, 10, Sort.by("id").descending());

Page<User> page = userRepository.findAll(pageRequest);
List<User> users = page.getContent(); // Сами данные
int totalPages = page.getTotalPages(); // Общее кол-во страниц
```