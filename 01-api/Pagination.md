# Пагінація в .NET/Azure

## Azure SDK: `Pageable<T>` / `AsyncPageable<T>`
Уся пагінація в Azure-SDK (блоби, Cosmos, таблиці, ресурси) стандартизована однією абстракцією:
- будь-який `list/query` повертає `AsyncPageable<T>`, і ти просто `await foreach` — SDK **прозоро тягне сторінки** під капотом через continuation-токени, ти їх не бачиш;
- потрібен контроль посторінково — `.AsPages(continuationToken, pageSizeHint)`: дає `Page<T>` із `.Values` **і** `.ContinuationToken`. Токен можна **зберегти, зупинитись і продовжити пізніше**, або **віддати клієнту** як курсор.

Тобто пагінація однакова через **усі** сервіси — вивчив раз. Continuation-токен тут = opaque серверний курсор.

## Cosmos DB: continuation-токени (а offset — дорогий)
Cosmos пагінується **тільки токенами**:
- `FeedIterator` + `ReadNextAsync()` → `FeedResponse<T>` із `.ContinuationToken`; токен кодує, **де саме зупинився запит** (стан по кожній фізичній партиції), і поверненням його ти продовжуєш **точно з того місця**;
- **`OFFSET/LIMIT` існує, але жере RU** (Request Units — валюта пропускної Cosmos): щоб віддати `OFFSET 10000`, Cosmos усе одно **прочитає й викине** пропущені елементи → глибокий offset дорогий і легко впирається в throttling;
- нюанс: токен **привʼязаний до конкретного запиту** (зміниш запит — токен недійсний) і несе в собі стан крос-партиційного fan-out.

**Висновок для Cosmos:** проєктуй API навколо continuation-токенів, **не** навколо номерів сторінок.

## EF Core (SQL/Postgres)
- **Offset:** `.Skip(n).Take(size)` → генерує `OFFSET/FETCH`. Ок для неглибокого;
- **Keyset руками:** `.Where(x => x.Id > last).OrderBy(x => x.Id).Take(size)` — по індексу;
- вбудованого «курсора» нема — keyset пишеш сам. І `CountAsync()` для total — це **окремий дорогий запит**, тож часто total просто **не віддають**.

## OData — пагінація «з коробки»
Якщо береш ASP.NET Core OData, отримуєш paging безкоштовно:
- `$top` / `$skip` / `$count` / `$orderby` / `$filter`;
- **server-driven paging:** задаєш `PageSize`, і OData сам віддає **`@odata.nextLink`** із `$skiptoken` — клієнт просто **йде за nextLink**, нічого не рахуючи.

Ціна — OData важкий і opinionated (метадані, свій діалект). Але якщо він у проєкті вже є — пагінацію не пишеш.

## Патерн: своє API поверх Cosmos
Найчистіше рішення — **твій «cursor» = continuation-токен Cosmos**, проброшений клієнту як opaque-рядок: клієнт повертає його наступним запитом, ти пхаєш його назад у `FeedIterator` і віддаєш далі. **Нуль власної логіки курсора** — Cosmos усе тримає в токені.

**Суть:** у .NET/Azure пагінація стандартизована **`AsyncPageable<T>` + continuation-токени** через усі SDK; **Cosmos** пагінується токенами нативно, а offset там **дорогий по RU**; **EF Core** — `Skip/Take` або keyset руками; а «безкоштовну» пагінацію в своєму API дає **OData** (`@odata.nextLink`).
