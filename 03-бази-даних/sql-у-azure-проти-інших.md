# SQL у Azure проти інших

## Що Azure має «по SQL» — цілий зоопарк

Головне, що плутають: у Azure не «одна SQL база», а кілька різних сервісів.

**1. Azure SQL Database** — флагман. PaaS, повністю керований, на **рушії SQL Server**, але **evergreen** — завжди найновіший рушій, версією ти не керуєш (немає проєкту «мігруємо на SQL Server 2022»). Усередині ще й варіанти:
- **Single database** vs **Elastic pool** (багато баз ділять спільний пул ресурсів — економія, коли баз багато з різним навантаженням);
- **моделі оплати:** DTU (все в одному пакеті, просто) vs **vCore** (окремо компʼют і сховище, контроль + можна пере-використати ліцензії через Azure Hybrid Benefit);
- **тіри:** General Purpose, **Business Critical** (локальні SSD, HA-репліки, низька латентність), **Hyperscale** (до ~100 ТБ, миттєві бекапи, read-репліки);
- **Serverless compute** — авто-скейл компʼюту і **авто-пауза в простої** (pay-per-use, ідеально для дев/нерівномірного навантаження).

**2. Azure SQL Managed Instance** — теж PaaS, але **майже 100% сумісність зі справжнім SQL Server** на рівні інстансу: SQL Agent, крос-базові запити, CLR, Service Broker, linked servers. Це **ціль для lift-and-shift** наявних SQL Server апок. Сидить між VM і Azure SQL Database.

**3. SQL Server on Azure VM (IaaS)** — повний SQL Server на віртуалці. Максимум контролю, максимум мороки з обслуговуванням. Коли треба доступ до ОС або фіча, якої нема в PaaS. По суті не «керований».

**4. Azure Database for PostgreSQL** — керований Postgres:
- **Flexible Server** — актуальний, з зональним HA, вікнами обслуговування, burstable-тірами, stop/start;
- **Cosmos DB for PostgreSQL** (колишній Citus) — **розподілений/шардований** Postgres для горизонтального масштабу (мультивузол).

(Є ще Azure Database for MySQL, але тут фокус на SQL Server і Postgres.)

## Azure SQL Database vs справжній MS SQL Server

Ключове: **той самий рушій, інша операційна модель.**

| | MS SQL Server (on-prem/VM) | Azure SQL Database |
|---|---|---|
| Рушій | той самий T-SQL | той самий, але **evergreen** (завжди свіжий) |
| Патчинг/бекапи/HA | **ти сам** (Always On будуєш руками) | **Microsoft** (авто-failover, geo-репліка, PITR-бекапи) |
| Масштаб | купуєш залізо | слайдер vCore / Hyperscale до 100 ТБ / serverless-пауза |
| Ліцензії | купуєш per-core (дорого) | вшито в ціну (або Hybrid Benefit) |
| Instance-фічі | усі | обмежені (SQL Agent, крос-база — тільки в Managed Instance) |
| Розумні фічі | ставиш сам | вбудовані: **automatic tuning**, Query Performance Insight, Defender for SQL |

Тобто Azure SQL = «SQL Server, з якого зняли всю рутину адміністрування, але забрали доступ до ОС і частину instance-фіч». Якщо апці ці фічі потрібні → **Managed Instance**.

## Azure SQL (SQL Server) vs PostgreSQL

Тут уже **різні рушії й діалекти**:

- **Мова/природа:** SQL Server = T-SQL, пропрієтарний (Microsoft). Postgres = опенсорс, ближчий до стандарту, шалено **розширюваний**.
- **Ціна/ліцензія:** Postgres — опенсорс, **ліцензії за рушій нема** → на Azure платиш лише за керований сервіс, часто **дешевше**. SQL Server тягне ліцензію (вшиту в ціну Azure SQL).
- **Екосистема:** SQL Server — рідний для .NET (EF Core first-class, найщільніша інтеграція з Azure). Postgres — улюбленець опенсорсу, але й у .NET чудово живе (EF Core + Npgsql).
- **Розширення (сила Postgres):** JSONB, масиви, **pgvector** (ембединги для AI!), **PostGIS** (геодані), кастомні типи. SQL Server відповідає columnstore, in-memory OLTP і щільною BI-інтеграцією.
- **Масштаб вшир:** SQL Server/Azure SQL масштабуються **вгору** (Hyperscale для величезних); справжній горизонтальний шардинг на боці Postgres — це **Cosmos DB for PostgreSQL (Citus)**, чистіша розподілена історія.

## Де Azure реально виграє

- **Managed Instance** — майже 100% сумісність із SQL Server як PaaS. Це **міграційна суперсила**, якої інші хмари для SQL Server-навантажень не дають так глибоко (AWS RDS for SQL Server бідніший).
- **Hyperscale** — розчеплені компʼют/сховище, 100 ТБ, миттєві бекапи, read-репліки.
- **Serverless auto-pause** — база сама засинає в простої, ти платиш за паузу копійки.
- **Passwordless-доступ через Entra ID / Managed Identity** — конектишся до Azure SQL **без пароля в конекшн-стрінгу** (та сама «автообліковка», що й для Key Vault). Величезний плюс безпеки.
- **Вбудований інтелект** — automatic tuning (ML сам крутить індекси/плани), Defender for SQL, Azure Monitor — усе з коробки.

## Що обирати (рішенєва рамка)

- .NET-шоп, найщільніша інтеграція з Azure, T-SQL, evergreen → **Azure SQL Database**.
- Переносиш наявний SQL Server з instance-фічами → **Managed Instance**.
- Опенсорс, чутливий до ціни, потрібні розширення (pgvector під AI, PostGIS) → **Azure Database for PostgreSQL Flexible Server**.
- Треба реляційний **шардинг вшир** → **Cosmos DB for PostgreSQL (Citus)** або Hyperscale.
- Потрібен доступ до ОС / непідтримувана фіча → **SQL Server on VM** (останній варіант).

## Де оверкіл / чесна межа

- **Managed Instance або Hyperscale під маленьку апку** — стрільба з гармати, дорого; вистачить Azure SQL Database GP або навіть serverless.
- **SQL Server on VM «щоб контролювати»** — здебільшого ти платиш адмініструванням за фічі, які тобі не потрібні; PaaS дешевший у супроводі.
- **Крос-хмара чесно:** на боці **Postgres** AWS Aurora — дуже сильний (розподілене сховище), тут паритет або й перевага AWS. А от на **SQL Server** Azure виграє впевнено (Managed Instance, Hyperscale, Entra-passwordless, EF Core).

---

**Суть:** Azure SQL Database — це SQL Server без адмінської рутини (evergreen + вбудовані HA/tuning/Defender), Managed Instance — міграційний міст для instance-фіч, а Postgres на Azure виграє ціною й розширеннями (pgvector/PostGIS).
