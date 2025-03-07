# Глава 1: Фундамент и Философия

## 1\. Основы JDBC и SQL (Зачем нам вообще ORM?)

Прежде чем радоваться магии Hibernate, нужно понять боль, которую он лечит.

### Проблема JDBC

В "голом" Java для выполнения одного запроса нужно:

1.  Открыть соединение (`Connection`).
2.  Создать выражение (`PreparedStatement`).
3.  Вставить параметры (защита от SQL Injection).
4.  Выполнить запрос.
5.  Пройтись циклом по `ResultSet`.
6.  Смапить колонки из `ResultSet` в поля Java-объекта вручную.
7.  Закрыть все ресурсы в блоке `finally` или `try-with-resources`.

**Пример боли (JDBC):**

```java
User user = new User();
try (Connection c = dataSource.getConnection();
     PreparedStatement ps = c.prepareStatement("SELECT * FROM users WHERE id = ?")) {
    ps.setLong(1, 1L);
    try (ResultSet rs = ps.executeQuery()) {
        if (rs.next()) {
            user.setId(rs.getLong("id"));
            user.setName(rs.getString("name"));
            // И так для каждого поля...
        }
    }
} catch (SQLException e) {
    throw new RuntimeException(e);
}
```

### Решение ORM

ORM (Object-Relational Mapping) автоматизирует пункты 2–6. Вы работаете с объектами, библиотека генерирует SQL.

-----

## 2\. Введение в JPA и Конфигурация

### JPA vs Hibernate

Это самый частый вопрос на собеседованиях.

  * **JPA (Jakarta Persistence API)** — это **спецификация** (набор интерфейсов и правил). Это как интерфейс `List` в Java.
  * **Hibernate** — это **реализация** этой спецификации. Это как `ArrayList`.
  * *Spring Data JPA* — это еще одна надстройка над JPA, упрощающая написание репозиториев (о ней позже).

### Базовые аннотации

Чтобы Hibernate понял, как связать класс с таблицей, используются аннотации из пакета `javax.persistence` (или `jakarta.persistence` в новых версиях).

```java
@Entity // Говорит, что этот класс — сущность БД
@Table(name = "users") // Имя таблицы (если отличается от имени класса)
public class User {

    @Id // Первичный ключ
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Стратегия генерации
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;
    
    // Геттеры, сеттеры, пустой конструктор (обязателен для Hibernate!)
}
```

### Стратегии генерации ID (`@GeneratedValue`)

1.  **AUTO**: Hibernate сам выбирает стратегию (опасно для продакшена, может вести себя непредсказуемо).
2.  **IDENTITY**: Использует автоинкремент базы данных (MySQL `AUTO_INCREMENT`, Postgres `SERIAL`).
      * *Минус:* Hibernate не может узнать ID, пока не сделает `INSERT`. Это отключает Batch-вставку (оптимизацию).
3.  **SEQUENCE**: Использует объект Sequence в БД (Oracle, Postgres).
      * *Плюс:* Самый производительный вариант. Позволяет узнать ID *до* вставки в БД.
4.  **TABLE**: Хранит счетчики в отдельной таблице. Медленно, использовать только при необходимости.

-----

## 3\. Жизненный цикл Entity и Persistence Context

Это **сердце** Hibernate. Если вы поймете это, вы поймете 80% проблем.

**Persistence Context (Контекст Персистентности)** — это "кэш первого уровня" (L1 Cache). Это область памяти, где Hibernate хранит объекты, которые он загрузил или сохраняет в рамках одной транзакции (Сессии).

### 4 состояния сущности (Entity States)

1.  **Transient (New):** Объект просто создан через `new`. Hibernate о нем ничего не знает. В БД его нет.
2.  **Managed (Persistent):** Объект находится в контексте. У него есть ID. Все изменения в полях этого объекта будут **автоматически** записаны в БД при завершении транзакции.
3.  **Detached:** Объект есть в БД, но сессия Hibernate уже закрыта или объект принудительно выкинут из контекста. Изменения в нем *не* сохранятся.
4.  **Removed:** Объект помечен на удаление. При коммите транзакции произойдет `DELETE`.

### Dirty Checking (Грязная проверка)

Новички часто пишут лишний код, вызывая `save()` каждый раз.

**Неправильно (в транзакции):**

```java
@Transactional
public void updateName(Long id, String newName) {
    User user = repository.findById(id).get(); // User стал Managed
    user.setName(newName);
    repository.save(user); // <--- ЭТО ЛИШНЕЕ!
}
```

**Правильно:**

```java
@Transactional
public void updateName(Long id, String newName) {
    User user = repository.findById(id).get(); // Загрузили в контекст
    user.setName(newName); 
    // Метод завершается -> Транзакция коммитится -> 
    // Hibernate видит, что поле изменилось -> делает UPDATE сам.
}
```

### Основные методы EntityManager

  * `persist(entity)`: Делает Transient -\> Managed. (Планирует INSERT).
  * `merge(entity)`: Делает Detached -\> Managed. (Загружает копию из БД, копирует поля из переданного объекта, возвращает managed-копию).
  * `remove(entity)`: Делает Managed -\> Removed.
  * `flush()`: Принудительно отправляет SQL команды из памяти в базу данных (но не делает Commit).

-----