# Разбор кода репозитория «The Last of Guss»

Проблемы отсортированы по критичности. Их удобно использовать как «крючки» в ответах на вопросы из [README.md](README.md).

## Сервер

### Критичные: нарушают требования из TASK.md

1. **При тапе не проверяется, что раунд активен.** В [games.service.ts:75-96](server/src/games/games.service.ts#L75-L96) `processTap` увеличивает `taps` без проверки `start_datetime <= now < end_datetime`. Тапать можно во время cooldown и после завершения раунда, поэтому победителя можно «докрутить» уже после конца.

2. **Тап падает, если строки `score` ещё нет.** В [games.service.ts:78](server/src/games/games.service.ts#L78) `increment` по несуществующей строке ничего не делает. Затем `findOne` возвращает `null`, и [games.service.ts:93](server/src/games/games.service.ts#L93) падает на `scoreRecord.taps` с ошибкой 500. Сейчас это работает только потому, что `GET /round/:uuid` создаёт запись заранее ([games.controller.ts:33](server/src/games/games.controller.ts#L33)). Получается побочный эффект в GET и неявная зависимость между эндпоинтами.

3. **Тап выполняется не атомарно.** `increment` и `findOne` — два отдельных запроса без транзакции и без проверки времени в том же запросе. Правильнее сделать один атомарный запрос по времени **БД**, а не Node.js: при 3 инстансах их часы могут расходиться.
   ```sql
   INSERT INTO scores ("user", round, taps)
   SELECT $1, r.uuid, 1 FROM rounds r
   WHERE r.uuid = $2 AND now() >= r.start_datetime AND now() < r.end_datetime
   ON CONFLICT ("user", round) DO UPDATE SET taps = scores.taps + 1
   RETURNING taps;
   ```
   Если запрос не вернул строк, значит раунд неактивен, и нужно ответить 409/403. Для Никиты выполняется тот же запрос без инкремента.

4. **Роль `admin` никогда не назначается.** В [auth.service.ts:33](server/src/auth/auth.service.ts#L33) роль бывает только `nikita` или `user`, поэтому создать раунд не может никто.

5. **Нет общего счётчика очков раунда.** По ТЗ при тапе нужно увеличивать и его. Вместо этого [games.service.ts:98-138](server/src/games/games.service.ts#L98-L138) загружает все строки `scores` в память и считает сумму и максимум в JS. Это можно сделать одним SQL-агрегатом (`SUM`, `ORDER BY taps DESC LIMIT 1`). Ничья при выборе победителя не обрабатывается.

### API и корректность

6. **«Не найдено» возвращается с кодом 200.** В [games.controller.ts:29-31](server/src/games/games.controller.ts#L29-L31) ответ строится как `{ error: 'Round not found' } as any`, а должен быть `NotFoundException`.
7. **Нет валидации входных данных.** Не используются `ParseUUIDPipe` и DTO с `ValidationPipe`. Если передать невалидный uuid, Postgres вернёт ошибку, и клиент получит 500.
8. **Слабая типизация.** В [games.controller.ts:27](server/src/games/games.controller.ts#L27) и [:67](server/src/games/games.controller.ts#L67) стоит `req: any`, а в [:56](server/src/games/games.controller.ts#L56) тип `req` описан вручную и неверно. Лучше сделать кастомный декоратор `@CurrentUser()`.
9. **`GET /rounds` отдаёт и завершённые раунды** ([games.controller.ts:21-23](server/src/games/games.controller.ts#L21-L23)), хотя по ТЗ нужны только активные и запланированные. Сортировки тоже нет.
10. **Поле `rounds.status` бесполезно.** При создании пишется `'scheduled'` ([games.service.ts:65](server/src/games/games.service.ts#L65)), и больше это значение не обновляется. Статус должен вычисляться из времени. Кроме того, `end_datetime` объявлен с `allowNull: true` ([round.model.ts:22-26](server/src/models/round.model.ts#L22-L26)).
11. **Конфигурация читается в обход DI.** `process.env` читается внутри метода при каждом вызове ([games.service.ts:55-56](server/src/games/games.service.ts#L55-L56)). Правильнее использовать `ConfigService` с валидацией при старте, как и требует ТЗ: «применяются при старте бекенда».

### Безопасность и аутентификация

12. **Небезопасный fallback для секрета JWT.** В [jwt.strategy.ts:12](server/src/auth/jwt.strategy.ts#L12) стоит `'your-secret-key'`. Если забыть задать env, токены сможет подделать любой.
13. **Роль зашита в JWT** ([auth.service.ts:47](server/src/auth/auth.service.ts#L47)). Если роль поменяется, это не отразится до истечения токена, а отозвать токен нельзя. `username` дублирует `sub`.

### Идентификаторы

14. **В схеме смешаны два подхода к ключам.** `rounds.uuid` — UUIDv4 ([round.model.ts:9-14](server/src/models/round.model.ts#L9-L14)): случайные значения дают фрагментацию B-tree индекса. `users` использует натуральный ключ `login` ([user.model.ts:14-20](server/src/models/user.model.ts#L14-L20)): это строка, в том числе кириллическая. Она уходит во FK `scores.user` ([score.model.ts:17-22](server/src/models/score.model.ts#L17-L22)) и в `sub` токена, поэтому переименовать пользователя нельзя.

## Клиент

15. **Токен хранится в `localStorage`** ([api.ts:17](client/src/services/api.ts#L17), [:86](client/src/services/api.ts#L86)), поэтому его можно украсть через XSS. Роль декодируется на клиенте ([api.ts:115-134](client/src/services/api.ts#L115-L134)). Это нормально только для UI, сервер всё равно проверяет права.
16. **После перезагрузки счёт показывает 0.** В [RoundPage.tsx:28](client/src/pages/RoundPage.tsx#L28) `setTapCount(0)` вызывается при загрузке, а сервер не отдаёт текущий счёт для активного раунда. Проверка `response.score > tapCount` ([:73](client/src/pages/RoundPage.tsx#L73)) — костыль вокруг гонки ответов.
17. **setState вызывается прямо в рендере** ([RoundPage.tsx:127-129](client/src/pages/RoundPage.tsx#L127-L129)), а в `<h1>` остался отладочный вывод `к={needReloadOnFinish}` ([:178](client/src/pages/RoundPage.tsx#L178)).
18. **Таймер идёт по часам клиента.** Сервер не отдаёт `serverTime`, поэтому при расхождении часов клиент покажет «активен», а сервер (после исправления п.1) отклонит тап.
19. **Тапы теряются.** `isTapping` блокирует новые тапы на всё время запроса плюс 100 мс ([RoundPage.tsx:66](client/src/pages/RoundPage.tsx#L66), [:79](client/src/pages/RoundPage.tsx#L79)). Связка `mousedown`/`click` через тот же флаг хрупкая, а тач-события не обрабатываются.
20. **Вся страница перерисовывается раз в секунду**, потому что `currentTime` хранится в state корневого компонента ([RoundPage.tsx:50-56](client/src/pages/RoundPage.tsx#L50-L56)). Таймер стоит вынести в отдельный компонент.
21. **Мелочи:**
    - у `useEffect` неполные зависимости ([:59-62](client/src/pages/RoundPage.tsx#L59-L62));
    - в макете статус называется «Cooldown», а в коде «Ожидание»;
    - `performTap` дублирует `tap` ([api.ts:81-83](client/src/services/api.ts#L81-L83)).

## Что сделано хорошо

- Сервер stateless, JWT не требует sticky-сессий, так что 3 инстанса за прокси будут работать.
- На `(user, round)` есть уникальный индекс ([score.model.ts:8-14](server/src/models/score.model.ts#L8-L14)), так что задел для upsert уже есть.
- Есть общий пакет `contract` с типами для клиента и сервера.
- Очки не хранятся, а вычисляются из `taps`: `floor(taps/11)*9 + taps`. Формула верная, и рассинхронизации taps и score не бывает.
