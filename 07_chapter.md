# Глава 7. Сложная архитектура БД: Multi-tenancy и Репликация

## 1\. Multi-tenancy (Мульти-арендность)

Представьте, что вы пишете CRM-систему (SaaS). Ею пользуются компании "Рога и Копыта" (Tenant A) и "Вектор" (Tenant B).
**Главное требование:** Данные одной компании **никогда** не должны попасть к другой.

### Три стратегии реализации

1.  **Discriminator Column (Общая схема):**
      * Все живут в одной таблице `users`. Добавляется колонка `tenant_id`.
      * *Плюс:* Дешево, легко делать бэкап, миграции простые.
      * *Минус:* Разработчик забыл добавить `WHERE tenant_id = ?` в одном запросе -\> утечка данных. **Hibernate решает это через `@TenantId` (в версии 6+) или фильтры.**
2.  **Separate Schema (Раздельные схемы):**
      * Одна БД, но разные схемы: `schema_tenant_a`, `schema_tenant_b`.
      * *Плюс:* Хорошая изоляция, можно делать бэкап отдельной схемы.
      * *Минус:* Сложно управлять миграциями (Flyway должен пройтись по 100 схемам).
3.  **Separate Database (Раздельные базы):**
      * У каждого клиента свой URL подключения.
      * *Плюс:* Полная физическая изоляция (VIP клиенты на быстром железе).
      * *Минус:* Очень дорого по ресурсам.

### Реализация в Hibernate (Separate Schema)

Hibernate имеет встроенную поддержку этого механизма. Вам нужно реализовать два интерфейса.

#### Шаг 1. Кто стучится? (`CurrentTenantIdentifierResolver`)

Нам нужно понять, какой клиент сейчас делает запрос (обычно извлекаем из JWT токена или поддомена `client.app.com`).

```java
@Component
public class TenantResolver implements CurrentTenantIdentifierResolver {

    @Override
    public String resolveCurrentTenantIdentifier() {
        // Достаем ID из ThreadLocal (куда его положил фильтр запросов)
        String tenantId = TenantContext.getCurrentTenant();
        return tenantId != null ? tenantId : "default_schema";
    }

    @Override
    public boolean validateExistingCurrentSessions() {
        return true;
    }
}
```

#### Шаг 2. Дай подключение (`MultiTenantConnectionProvider`)

Hibernate просит: "Дай мне Connection для клиента 'tenant\_a'". Вы должны переключить схему.

```java
@Component
public class SchemaConnectionProvider implements MultiTenantConnectionProvider {

    @Autowired
    private DataSource dataSource;

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        Connection connection = dataSource.getConnection();
        // Магия Postgres: переключаем search_path на нужную схему
        connection.setSchema(tenantIdentifier); 
        return connection;
    }
    
    // ... остальные методы releaseConnection и т.д.
}
```

#### 🪨 Подводные камни (Expert)

1.  **Connection Pooling:** Если вы используете стратегию **Separate Database**, вы не можете создать 100 пулов соединений (HikariCP) по 10 коннектов. 1000 открытых сокетов положат сервер. Придется использовать динамическую маршрутизацию без пулинга или внешние прокси (PgBouncer).
2.  **Кэш Hibernate:** L2 Cache должен знать о тенантах. Иначе Tenant A получит кэшированные данные Tenant B. В EhCache/Redis это решается добавлением TenantID в ключ кэша.

-----

## 2\. Read/Write Splitting (Репликация)

Когда `SELECT` запросов становится слишком много, базу разделяют:

  * **Master (Leader):** Принимает `INSERT`, `UPDATE`, `DELETE`.
  * **Replica (Follower/Slave):** Принимает только `SELECT`. Данные копируются с Мастера асинхронно.

**Задача:** Заставить Spring автоматически отправлять пишущие запросы на Мастера, а читающие — на Реплику.

### Реализация: `AbstractRoutingDataSource`

В Spring есть специальный класс `AbstractRoutingDataSource`, который притворяется одним DataSource, но внутри переключается между несколькими реальными.

```java
// 1. Определяем типы
public enum DataSourceType { READ_ONLY, READ_WRITE }

// 2. Логика маршрутизации
public class TransactionRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        // Проверяем: текущая транзакция ReadOnly?
        if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            return DataSourceType.READ_ONLY;
        }
        return DataSourceType.READ_WRITE;
    }
}
```

**Конфигурация:**

1.  Создаем два реальных DataSource (MasterDS, ReplicaDS).
2.  Скармливаем их нашему `TransactionRoutingDataSource`.
3.  В сервисах используем аннотацию:

<!-- end list -->

```java
@Service
public class UserService {

    @Transactional(readOnly = true) // Пойдет на Replica
    public User getUser(Long id) { ... }

    @Transactional // (readOnly = false) Пойдет на Master
    public void createUser(User user) { ... }
}
```

### 🪨 Главный подводный камень: Replication Lag

Это классическая проблема распределенных систем.

**Сценарий:**

1.  Пользователь обновляет профиль (`Master DB`). Транзакция завершена.
2.  Пользователь сразу перенаправляется на страницу профиля (`SELECT` -\> `Replica DB`).
3.  Репликация занимает 100-500мс. Данные еще не долетели до Реплики.
4.  **Результат:** Пользователь видит старые данные. Он паникует и нажимает F5.

**Решения:**

1.  **Грубое:** После записи делать принудительное чтение с Мастера для этого пользователя (хранить флаг в сессии).
2.  **Архитектурное:** Критичные данные (профиль сразу после редактирования) всегда читать с Мастера (`@Transactional(readOnly = false)` даже для SELECT). Некритичные (лента новостей) — с Реплики.

-----

**Итог главы:**
Вы узнали, как масштабировать приложение горизонтально.

  * **Multi-tenancy** позволяет продавать один инстанс приложения сотням клиентов.
  * **Replication** позволяет выдерживать огромные нагрузки на чтение.