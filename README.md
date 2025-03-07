# Spring Data JPA и Hibernate: От Junior до Architect

Этот репозиторий содержит учебник по работе с реляционными базами данных в Java-экосистеме — от основ JPA и Hibernate до архитектурных паттернов, продвинутой оптимизации и hardcore-конкурентности.

## Описание

Курс рассчитан на разные уровни подготовки:

* **Junior** — поймете, зачем нужен ORM, как работает `EntityManager`, что такое Persistence Context и как маппить простые сущности.
* **Middle** — научитесь проектировать отношения между таблицами, использовать Spring Data JPA, писать запросы и настраивать транзакции.
* **Senior** — освоите продвинутый маппинг, Criteria API, Projections, оптимизацию запросов и кэширование.
* **Expert** — разберетесь в N+1, Batch Processing, Attribute Converters, Blaze-Persistence и аналитических запросах.
* **Architect** — спроектируете Multi-tenant системы, настроите репликацию Read/Write, разберетесь в уровнях изоляции, блокировках и миграциях.
* **Hardcore** — погрузитесь в Propagation, deadlocks, pool starvation и внутреннее устройство транзакционной модели.
* **Best Practices** — получите готовый рецепт идеальной Entity, шпаргалку по аннотациям и практики тестирования.

## Содержание

Учебник состоит из 12 глав, объединенных в уровни:

1. **Уровень 1: Фундамент и Философия** — основы JDBC и SQL, введение в JPA и Hibernate, жизненный цикл Entity и Persistence Context.
2. **Уровень 2: Отношения и Базовый Spring Data** — маппинг ассоциаций, Spring Data JPA, Derived Queries, пагинация.
3. **Уровень 3: Продвинутые возможности и Типы** — встраиваемые объекты, наследование, Criteria API, Specifications, Projections.
4. **Уровень 4: Производительность и Оптимизация** — проблема N+1, кэширование L1/L2, batch processing, прокси и LazyInitializationException.
5. **Уровень 5: Конкурентность, Транзакции и Архитектура** — управление транзакциями, блокировки, миграции (Flyway/Liquibase), тестирование.
6. **Уровень 6: Тонкая настройка данных** — Attribute Converters, Soft Delete, Entity Listeners.
7. **Уровень 7: Сложная архитектура БД** — Multi-tenancy, Read/Write Splitting, репликация.
8. **Уровень 8: Продвинутые запросы и Аналитика** — Native Queries, Blaze-Persistence, продвинутые Projections, дедупликация в Hibernate 6.
9. **Уровень 9: Конкурентность** — Propagation, Deadlocks, Pool Starvation.
10. **Уровень 10: Тестирование** — практика написания интеграционных тестов репозиториев.
11. **Уровень 11: Полезные аннотации** — шпаргалка по ключевым аннотациям JPA, Hibernate и Lombok.
12. **Уровень 12: Идеальная Entity** — рецепт production-ready сущностей на основе лучших практик.

Подробное оглавление доступно в файле [`00_toc.md`](00_toc.md).

## Актуальная сборка

Актуальную сборку учебника в формате PDF можно получить на странице релизов:

👉 [https://github.com/dmitry-osin/spring-data-jpa-book/releases](https://github.com/dmitry-osin/spring-data-jpa-book/releases)

## Лицензия

Этот проект распространяется под лицензией **MIT**.

Автор: **Dmitry Osin**

Подробности см. в файле [LICENSE](LICENSE).
