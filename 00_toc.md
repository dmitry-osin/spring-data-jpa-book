### Уровень 1: Фундамент и Философия

Прежде чем писать код, важно понять, какую проблему решает ORM.

* **1. Основы JDBC и SQL (Зачем нам вообще ORM?)**
* Проблема JDBC
* Решение ORM
* **2. Введение в JPA и Конфигурация**
* JPA vs Hibernate
* Базовые аннотации
* Стратегии генерации ID (`@GeneratedValue`)
* **3. Жизненный цикл Entity и Persistence Context**
* 4 состояния сущности (Entity States)
* Dirty Checking (Грязная проверка)
* Основные методы EntityManager

---

### Уровень 2: Отношения и Базовый Spring Data

Здесь вы учитесь связывать таблицы и упрощать доступ к данным.

* **1. Маппинг отношений (Associations)**
* Владеющая сторона (Owning Side) vs Обратная сторона (Inverse Side)
* Пример: One-To-Many (Один ко Многим)
* Каскадные операции (Cascade) и Orphan Removal
* **2. Знакомство со Spring Data JPA**
* Иерархия интерфейсов
* Магия Derived Queries (Запросы из имени метода)
* @Query (Когда имя метода слишком длинное)
* Пагинация и Сортировка

---

### Уровень 3: Продвинутые возможности и Типы

Переход от простых CRUD операций к сложным сценариям.

* **1. Продвинутый маппинг**
* Встраиваемые объекты (Value Objects)
* Маппинг Enums
* Наследование (Inheritance)
* **2. Продвинутое построение запросов**
* Criteria API
* Spring Data Specifications
* Projections (Проекции)

---

### Уровень 4: Производительность и Оптимизация

Это самый важный раздел для эксперта. Здесь вы учитесь не «валить» базу данных.

* **1. Проблема N+1 и Стратегии выборки (Fetching)**
* Что такое N+1?
* Решение 1: JOIN FETCH (JPQL)
* Решение 2: @EntityGraph (Spring Data)
* Решение 3: Hibernate Batch Fetching
* FetchType: EAGER vs LAZY
* **2. Кэширование (L1 и L2)**
* L1 Cache (Session Cache)
* L2 Cache (Shared Cache)
* **3. Оптимизация записи (Batch Processing)**
* **4. Прокси и LazyInitializationException**
* Почему возникает?
* Как работает Proxy?
* Как лечить LazyInit?

---

### Уровень 5: Конкурентность, Транзакции и Архитектура

Понимание того, как приложение ведет себя под нагрузкой и как его поддерживать.

* **1. Управление транзакциями**
* Аннотация @Transactional
* Propagation (Распространение)
* Уровни изоляции (Isolation Levels)
* **2. Блокировки (Locking)**
* Optimistic Locking (Оптимистичная блокировка)
* Pessimistic Locking (Пессимистичная блокировка)
* **3. Миграции баз данных (Schema Management)**
* Проблема `ddl-auto`
* Решение: Flyway или Liquibase
* **4. Тестирование**
* @DataJpaTest
* H2 vs Testcontainers

---

### Уровень 6: Тонкая настройка данных: Конвертеры и Soft Delete

Тонкая настройка хранения данных: от шифрования до мягкого удаления.

* **1. Attribute Converters (`@Converter`)**
* Как это работает
* Реализация (Шифрование данных)
* 🪨 Подводные камни
* **2. Soft Delete (Мягкое удаление)**
* Реализация (Современный подход Hibernate 6.3+)
* Как это выглядит в коде
* 🪨 Подводные камни (Expert Level)
* **3. Entity Listeners (Обратные вызовы)**
* 🪨 Важное ограничение

---

### Уровень 7: Сложная архитектура БД: Multi-tenancy и Репликация

Архитектурные паттерны для сложных многопользовательских систем.

* **1. Multi-tenancy (Мульти-арендность)**
* Три стратегии реализации
* Реализация в Hibernate (Separate Schema)
* **2. Read/Write Splitting (Репликация)**
* Реализация: `AbstractRoutingDataSource`
* 🪨 Главный подводный камень: Replication Lag

---

### Уровень 8: Продвинутые запросы и Аналитика

Выход за рамки стандартного JPA: аналитика, проекции и современный SQL.

* **1. Проблема чистого JPA и Native Queries**
* Решение 1: Native Query (Лобовая атака)
* 🪨 Подводные камни
* **2. Blaze-Persistence (Секретное оружие)**
* Пример: CTE (Иерархия категорий)
* **3. Продвинутые Проекции (Projections)**
* Уровень 1: Интерфейсы (Было выше)
* Уровень 2: Class-based (DTO) — Самый быстрый
* Уровень 3: Динамические проекции
* 🪨 Подводный камень (Dynamic Projections)
* **4. Hibernate 6: Дедупликация**

---

### Уровень 9: Конкурентность: Propagation, Deadlocks и Pool Starvation

Глубокое погружение в транзакции, блокировки и работу пула соединений.

* **1. Propagation (Распространение транзакций)**
* Ключевые стратегии
* ☠️ Главная ловушка: Self-Invocation (Вызов самого себя)
* **2. Deadlocks (Взаимоблокировки в БД)**
* Классический пример: Перевод денег
* Решение: Детерминированный порядок блокировок
* **3. Pool Starvation (Голодание пула)**

---

### Уровень 10: Тестирование

Практика: как правильно тестировать репозитории.

* **1. Зависимости (pom.xml)**
* **2. Базовый класс конфигурации**
* **3. Сам тест (UserRepositoryTest)**
* **На что обратить внимание в этом коде:**

---

### Уровень 11: Полезные аннотации Spring Data JPA и Hibernate

Шпаргалка по ключевым аннотациям экосистемы.

* **1. Основные (Identity & Setup)**
* `@Entity` (JPA)
* `@Table(name = "...")` (JPA)
* `@Id` (JPA)
* `@GeneratedValue` (JPA)
* **2. Маппинг полей (Basic Mapping)**
* `@Column` (JPA)
* `@Enumerated` (JPA)
* `@Transient` (JPA)
* `@Lob` (JPA)
* **3. Связи (Relationships)**
* `@OneToMany` / `@ManyToOne` (JPA)
* `@JoinColumn` (JPA)
* `@ManyToMany` (JPA)
* **4. Hibernate Specific (Power Tools)**
* `@BatchSize(size = 20)` (Hibernate)
* `@Formula("sql query")` (Hibernate)
* `@DynamicInsert` / `@DynamicUpdate` (Hibernate)
* `@Immutable` (Hibernate)
* **5. Аудит и Версионирование**
* `@Version` (JPA)
* `@CreationTimestamp` / `@UpdateTimestamp` (Hibernate)
* **6. ☠️ Особая зона: Lombok**
* `@Data` (Lombok)

---

### Уровень 12: Идеальная Entity

Рецепт идеальной сущности на основе лучших практик.

* **1. Базовый класс (Auditing & MappedSuperclass)**
* **2. Идеальная Сущность (User)**
* **3. Вторая сторона (Order)**
* **Почему именно так? (Разбор ключевых решений)**
* Почему `hashCode` возвращает константу?
* Почему `Hibernate.getClass(this)`?
* Почему `@BatchSize`?
* Почему `allocationSize = 50`?

---
