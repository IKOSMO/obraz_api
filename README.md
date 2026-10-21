# API-документация Obraz — PROD

> **Источник:** прод-сервер `185.93.111.249`, снято 2026-10-20.
> Маршруты, обработчики, поля запроса и коды взяты из активных исходников; схемы ответов — из функций БД; часть маршрутов дополнительно вызвана на живом проде.

## Содержание

- [1. Общие правила](#1-общие-правила)
- [2. Авторизация: токены](#2-авторизация-токены)
- [3. Пользователи (user_service)](#3-пользователи-user_service)
- [4. Гараж (cars_service)](#4-гараж-cars_service)
- [5. Мессенджер (messenger_service)](#5-мессенджер-messenger_service)
- [6. Контент (content_service)](#6-контент-content_service)
- [7. Помощь (help_service)](#7-помощь-help_service)
- [8. Публичные префиксы шлюза](#8-публичные-префиксы-шлюза)
- [9. Сервисы без исходников](#9-сервисы-без-исходников)
- [10. Найденные дефекты](#10-найденные-дефекты)

---

## 1. Общие правила

### Аутентификация

Защищённые маршруты живут под `/v1/protected/...`. Схема:

```
клиент → Authorization: Bearer <accessToken> → nginx-gateway
                                     ↓ auth_request /auth_verify
                          auth_service /v1/auth/token/verify
                                     ↓ 200 + X-User-Id
           nginx подставляет X-User-Id → сервис
```

- Клиент отправляет только `Authorization: Bearer <token>`. Заголовок `X-User-Id` выставляет шлюз.
- Если отправить `X-User-Id` вручную — шлюз его перезапишет своим значением.
- Без токена / с неверным: `401` `{"message":"Отсутствует токен или неверный формат"}`.
- Внутренние вызовы между сервисами идут через `/internal/...` — снаружи недоступны.

### Формат запросов

| Параметр | Значение |
|---|---|
| Тело | JSON, `Content-Type: application/json` |
| Исключение | `POST /users/avatar` — сырые байты изображения |
| Исключение | `GET /messenger/chat` — WebSocket |
| Пути | `<user_id>`, `<car_id>` и др. — целые числа |
| Тело читается как `Map` без типизации | поля не валидируются схемой |

### Формат ответов

Единого формата ошибок нет. Три варианта:

| Формат | Пример | Доля |
|---|---|---|
| Текст исключения | `error: Bad state: No element` | большинство `500` |
| JSON `{"message": ...}` | `{"message":"Невозможно..."}` | часть `400`/`401` |
| Без тела | `204`, `400` | валидация, пустые списки |

**Следствие для клиента:** на `500` приходит внутренний текст Dart с типами и деталями БД. Стабильно разбирать такие ответы нельзя — обрабатывайте по коду статуса.

### Коды

| Код | Когда |
|---|---|
| 200 | Успех |
| 201 | Создано |
| 204 | Успех, результат пуст |
| 400 | Валидация / нет поля |
| 401 | Нет токена / неверные данные |
| 403 | Нет прав |
| 500 | Внутренняя ошибка |

### Списки приходят объектами, не массивами

Коллекции отдаются как `{"0": {...}, "1": {...}}` — не JSON-массив. Порядок ключей не гарантирован.

---

## 2. Авторизация: токены

Источник: исходники `auth_service` (`bin/server.dart`, `lib/jwt.dart`, `lib/config.dart`) и проверка живого прода 2026-10-21.
Прод запускает образ `registry.gitlab.com/obraz2/auth_service:latest` — сборка может отличаться от исходников, поэтому ниже отдельно помечено, что подтверждено вызовом, а что известно только по коду.

### Схема

```text
клиент ── login / signup ──► user_service
                                │  внутренний вызов
                                ▼
                    GET /v1/auth/token/release  (заголовок X-User-Id)
                                │
                                ├─► accessToken  — JWT (HS256), 60 минут
                                └─► refreshToken — UUIDv4 → Redis, TTL 24 часа

клиент ── Authorization: Bearer <accessToken> ──► nginx-gateway
                                                        │ auth_request
                                                        ▼
                                        GET /v1/auth/token/verify
                                                        │ 200 + X-User-Id
                                    nginx подставляет X-User-Id → сервис
```

Клиент работает только с `accessToken` и `refreshToken`. `X-User-Id` снаружи не передаётся — его выставляет шлюз по результату `verify`.

### Access-токен

| Свойство | Значение |
|---|---|
| Тип | JWT, подпись HS256 |
| Секрет | `JWT_SECRET` из окружения сервиса |
| Payload | `{"user_id": <int>, "exp": <unix-время>}` |
| Срок жизни | **60 минут** (`Duration(minutes: 60)` в `generateTokens` и `refreshTokens`) |
| Хранение на сервере | нет, токен stateless |
| Отзыв | **невозможен** — действителен до истечения `exp` |

`user_id` в токене — число, не строка. Это исправление: строка `"26"` приводила к `'String' is not a subtype of type 'int'` в клиенте на Dart (комментарий в `_releaseTokenHandler`).

### Refresh-токен

| Свойство | Значение |
|---|---|
| Тип | случайный UUIDv4, не JWT |
| Хранение | Redis, ключ `refreshToken:<uuid>`, значение — `user_id` |
| Срок жизни | **24 часа** (`refreshTokenTTL = 3600 * 24`) |
| Привязка к устройству | нет |
| Ротация | нет (см. ниже) |
| Отзыв | эндпоинта нет |

### Маршруты

Все три обслуживает `auth_service` на порту `8080` внутри сети Swarm. Снаружи доступен только `/v1/auth/*` через шлюз; `/token/verify` и `/token/release` рассчитаны на внутренние вызовы.

| Маршрут | Метод | Вход | Ответ |
|---|---|---|---|
| `/v1/auth/token/release` | GET | `X-User-Id` (внутренний вызов) | `200` `{accessToken, refreshToken, user_id}` |
| `/v1/auth/token/verify` | GET | `Authorization: Bearer <accessToken>` | `200` + заголовок `X-User-Id`, тело пустое |
| `/v1/auth/token/refresh` | POST | `{"refreshToken": "<uuid>"}` | `200` `{accessToken, refreshToken}` |
| `/v1/auth/token/revoke` | — | — | **отсутствует** в коде и в бинарнике прода |

`release` доступен только изнутри: без `X-User-Id` отдаёт `400`.

### Обновление (refresh)

```
POST /v1/auth/token/refresh
Content-Type: application/json

{"refreshToken": "<uuid>"}
```

Логика обработчика:

1. Читает `refreshToken` из тела.
2. Ищет его в Redis: `GET <refreshToken>`.
3. Если ключа нет — `401` `{"message":"Недействительный refresh токен"}`.
4. Если ключ есть, берёт из значения `user_id` и выпускает **новый access-токен на 60 минут**.
5. Возвращает `{accessToken, refreshToken}`.

Статусы: `200`, `401`, `500` (при исключении — текст ошибки Dart в теле).

### Ротация

Ротации нет.

`refreshTokens()` возвращает только новый `accessToken` — нового `refreshToken` в ответе нет. Обработчик подставляет `tokens['refreshToken']` в тело ответа, но по этой ветке значение будет `null`: клиент получает `{"accessToken": "...", "refreshToken": null}`.

Старый refresh-токен при обновлении **не удаляется** из Redis. Он остаётся действительным до истечения собственных 24 часов, и его можно использовать повторно неограниченное число раз.

### Отзыв

Механизма отзыва нет ни для одного типа токена:

- **Access-токен** отозвать нельзя — он stateless, проверяется только подпись и `exp`. Выход из аккаунта, смена пароля и блокировка пользователя не делают его недействительным: он живёт до конца своих 60 минут.
- **Refresh-токен** отозвать через API нельзя — эндпоинта `revoke` нет. Единственный способ — вручную удалить ключ в Redis:
  ```
  docker exec <redis> redis-cli DEL "refreshToken:<uuid>"
  ```
- **«Выйти на всех устройствах»** не реализовано: у пользователя может быть сколько угодно параллельных refresh-токенов, и все они независимы.

### Проверено на живом проде 2026-10-21

Вызовы выполнялись изнутри кластера, без выпуска настоящих токенов.

| Запрос | Ответ |
|---|---|
| `GET /v1/auth/token/verify` без `Authorization` | `401` `{"message":"Отсутствует токен или неверный формат"}` |
| `GET /v1/auth/token/verify` с `Bearer abc.def.ghi` | `401` `{"message":"Ошибка проверки JWT","error":"JWTUndefinedException: FormatException: Unexpected extension byte (at offset 1)"}` |
| `POST /v1/auth/token/refresh` с несуществующим UUID | `401` `{"message":"Недействительный refresh токен"}` |
| `GET /v1/auth/token/release` без `X-User-Id` | `400` `{"error":"X-User-Id required for token release"}` |
| `auth_service` на порту `8000` | недоступен, работает только `8080` |

Успешный сценарий (выпуск пары и последующий refresh) на проде не прогонялся — для этого требуется учётная запись. См. пункт 1 в разделе рисков.

### Риски и расхождения

1. **Несовпадение ключа Redis между `release` и `refresh`.** `release` пишет `refreshToken:<uuid>`, а `refresh` читает просто `<uuid>` без префикса (`redis.send_object(['GET', refreshToken])`). При таком коде refresh должен всегда отвечать `401`. Живым вызовом это не подтверждено — нужен тест на тестовой учётной записи. Если подтвердится, обновление токена на проде не работает и сессия живёт максимум 60 минут.
2. **Нет ротации refresh-токена.** Даже при рабочем refresh старый токен остаётся валидным все 24 часа, а новый не выдаётся (`refreshToken: null` в ответе).
3. **Нет отзыва.** Украденный refresh-токен действует 24 часа, украденный access-токен — до 60 минут. Досрочно прервать доступ можно только правкой Redis.
4. **60 минут на access-токен — много** при отсутствии отзыва и привязки к устройству.
5. **Refresh-токен не привязан к устройству, IP и `device_token`.** Поле `device_token` принимается в `signup`/`login`, но на токены не влияет.
6. **Секреты.** Stage читает `JWT_SECRET` из окружения и падает при старте, если переменная не задана (fail-fast, прод-значения по умолчанию из кода удалены). Прод использует свою конфигурацию. Значения секретов в документ не выносятся.
7. **Отладочный вывод.** Обработчики печатают в лог весь `Authorization`, тело запроса и сам `refreshToken` (`print("refreshToken = $refreshToken")`). Токены попадают в логи сервиса в открытом виде.

### Рекомендации

- Привести ключ в `refresh` к тому же виду, что пишет `release` (`refreshToken:<uuid>`), либо убрать префикс в обоих местах.
- При обновлении выдавать новый refresh-токен и удалять старый — это делает кражу refresh-токена одноразовой.
- Добавить `POST /v1/auth/token/revoke`: удаление ключа из Redis плюс инвалидация access-токенов через счётчик версий у пользователя (`token_version` в payload).
- Сократить access-токен до 10–15 минут, чтобы окно компрометации было меньше.
- Убрать печать токенов в лог.

---

## 3. Пользователи (user_service)

Маршрутов: 26

### `POST /v1/users/signup`

- **Авторизация:** нет (публичный)
- **Тело запроса:**

```json
{"bdate", "description", "device_token", "email", "fullname", "license_plate", "license_plate_visibility", "password", "phone", "username"}
```

- **Коды:** 500
- **Ответ (объект):** `accessToken`, `refreshToken`, `user_id` (int)

### `POST /v1/users/login`

- **Авторизация:** нет (публичный)
- **Тело запроса:**

```json
{"device_token", "email", "password", "phone", "username"}
```

- **Коды:** 500
- **Ответ (объект):** `accessToken`, `refreshToken`, `user_id` (int)

### `POST /v1/users/check`

- **Авторизация:** нет (публичный)
- **Тело запроса:**

```json
{"email", "number", "username"}
```

- **Коды:** 200, 400, 500
- **Ответ (объект):** `{phone}` или `{email}`

### `POST /v1/users/newpass`

- **Авторизация:** нет (публичный)
- **Тело запроса:**

```json
{"email", "number", "password", "username"}
```

- **Коды:** 200, 400, 500
- **Ответ (пусто):** тела нет

### `POST /v1/protected/users`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"id_list"}
```

- **Коды:** 204, 500
- **Ответ (объект):** ключ = id: `{username, avatar}`

### `POST /v1/protected/users/`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"id_list"}
```

- **Коды:** 204, 500
- **Ответ (объект):** то же, что `POST /users`

### `POST /v1/protected/users/sub`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 204, 500
- **Ответ (объект):** ключи `1..N`: `{id, username, avatar}`

### `POST /v1/protected/users/change`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** читается как JSON, поля в коде не выделены
- **Коды:** 200, 400, 500
- **Ответ (пусто):** тела нет

### `POST /v1/protected/users/avatar`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** сырые байты изображения
- **Коды:** 200, 500
- **Ответ (пусто):** тела нет

### `GET /v1/protected/users`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (объект):** `user_id` (string), `username`, `fullname`, `description`, `sex`, `avatar` (string|null), `rating` (string), `subscribers` (string), `subscribes` (string)
- **Проверено на проде:** `200`

```json
{"user_id":"212","username":"shtabels","fullname":"Сергей Штабс","description":"Developer","sex":"1","avatar":"https://...","rating":"19","subscribers":"6","subscribes":"13"}
```

### `POST /v1/protected/users/subscribe`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"subscribe_to_id"}
```

- **Коды:** 201, 500
- **Ответ (пусто):** тела нет, код 201

### `GET /v1/protected/users/subscriptions`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `1..N`: `{user_id, username, fullname, avatar}`
- **Проверено на проде:** `200`

```json
{"1":{"user_id":115,"username":"katya_","fullname":"Екатерина","avatar":"https://..."}}
```

### `GET /v1/protected/users/subscribers`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `1..N`: `{user_id, username, fullname, avatar}`

### `GET /v1/protected/users/friends`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `1..N`: `{id, username, avatar}`
- **Проверено на проде:** `500`

```
error: Converting object to an encodable object failed
```

### `GET /v1/protected/users/rating`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `1..N`: `{user_id, username, fullname, avatar, likes, comments, subs, total_score, rating, you (bool)}`
- **Проверено на проде:** `200`

```json
{"1":{"user_id":312,"username":"Liliya_Chernykh","fullname":"Лилия Черных","avatar":"https://...",...}}
```

### `POST /v1/protected/users/settings`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"license_plate_visibility"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** тела нет

### `GET /v1/protected/users/settings`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (объект):** `user_id` (int), `license_plate_visibility` (int)
- **Проверено на проде:** `200`

```json
{"user_id":212,"license_plate_visibility":0}
```

### `POST /v1/protected/users/get_settings`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"id_list"}
```

- **Коды:** 200, 500
- **Ответ (объект):** ключ = `user_id`: `{license_plate_visibility (int)}`

### `GET /v1/protected/users/<user_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `user_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (объект):** `id` (int), `username`, `fullname`, `avatar` (string|null), `description`, `subscribers` (string), `subscribes` (string), `rating` (string), `subscribed` (bool)
- **Проверено на проде:** `200`

```json
{"id":212,"username":"shtabels","fullname":"Сергей Штабс","avatar":"https://...","description":"Developer","subscribers":"6","subscribes":"13","rating":"19","subscribed":false}
```

### `GET /v1/protected/users/friends/<user_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `user_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `1..N`: `{id, username, avatar}`

### `GET /v1/protected/users/subscribers/<user_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `user_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `1..N`: `{user_id, username, fullname, avatar}`

### `GET /v1/protected/users/subscriptions/<user_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `user_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `1..N`: `{user_id, username, fullname, avatar}`

### `GET /v1/protected/users/find/`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** то же, что `/users/find/<username>`, но список всех

### `GET /v1/protected/users/find/<username>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `username` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `1..N`: `{id, username, avatar, description, has_stories (bool), common_friends}`
- **Проверено на проде:** `500`

```
error: Bad state: No element
```

### `GET /v1/protected/users/license_plate_visibility`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** `id_list` — список `user_id` с видимостью номера

### `POST /v1/protected/users/find/license_plate`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"id_list", "license_plate"}
```

- **Коды:** 200, 204, 500
- **Ответ (объект):** `id_list`, `users`; либо 204

---

## 4. Гараж (cars_service)

Маршрутов: 15

### `POST /v1/protected/garage/vin/check`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"vin"}
```

- **Коды:** 200, 204, 500
- **Ответ:** 200, если VIN свободен; 204, если занят

### `GET /v1/protected/garage/makes`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (объект):** ключи `0..N`: `{id, name, country}`
- **Проверено на проде:** `200`

```json
{"0":{"id":1,"name":"Abarth","country":"Италия"},"1":{"id":2,"name":"AC","country":"Великобритания"}}
```

### `GET /v1/protected/garage/models/<make_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `make_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (объект):** ключи `0..N`: `{id, name, year_from, year_to, make_id}`

### `POST /v1/protected/garage/body_types`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (объект):** ключи `0..N`: `{id, name}`

### `POST /v1/protected/garage/modifications`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"body_type_id", "make_id", "model_id", "year"}
```

- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `0..N`: `{id, make_id, model_id, body_type_id, drive_type, fuel_type, is_hybrid, transmission, engine_capacity, power}`

### `POST /v1/protected/garage`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"body_type_id", "cng", "color", "ctc", "description", "drive_type", "engine_capacity", "fuel_type", "hybrid", "license_plate", "make_id", "media", "media_ctc", "mileage", "model_id", "modification_id", "no_mileage_in_russia", "power", "steering_wheel", "transmission", "vin", "year"}
```

- **Коды:** 201, 403, 500
- **Ответ (пусто):** 201 при успехе, 403 — нет прав

### `GET /v1/protected/garage/<user_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `user_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** ключи `0..N`: `{id, vin, ctc_number, license_plate, make_id, make_name, model_id, model_name, steering_wheel, year_of_release, body_id, body_name, drive_type, fuel_type, hybrid, transmission, engine_capacity, power, cng, mileage, no_mileage_in_russia, color, description, media, user_id, status}`
- **Проверено на проде:** `200`

```json
{"0":{"id":300,"vin":null,"ctc_number":null,"license_plate":"A324AA.324","make_id":2,"make_name":"AC",...}}
```

### `POST /v1/protected/garage/car/edit`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"body_type_id", "cng", "color", "ctc", "description", "drive_type", "engine_capacity", "fuel_type", "hybrid", "id", "license_plate", "make_id", "media", "media_ctc", "media_links_list", "mileage", "model_id", "modification_id", "no_mileage_in_russia", "power", "steering_wheel", "transmission", "vin", "year"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `DELETE /v1/protected/garage/car/<car_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `car_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `POST /v1/protected/garage/car/confirm`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"ctc_number", "id", "license_plate", "media_ctc", "vin"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `GET /v1/protected/garage/find/`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** то же, что `/garage/find/<match_text>`

### `GET /v1/protected/garage/find/<match_text>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `match_text` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** `id_list`, `users`; ключи `0..N` — как у `/garage/<user_id>` + `username`, `avatar`, `user_id`, `status`
- **Проверено на проде:** `200`

```json
{"0":{"id":207,"vin":null,"license_plate":"T685OO.25","make_id":344,"make_name":"Toyota",...}}
```

### `GET /v1/protected/garage/find/by_license_plate/`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** то же

### `GET /v1/protected/garage/find/by_license_plate/<license_plate>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `license_plate` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** `cars`, `users`, `id_list` + поля машины

### `GET /v1/protected/garage/check/<license_plate>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `license_plate` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** `id_list` пользователей с таким номером, либо 204

---

## 5. Мессенджер (messenger_service)

Маршрутов: 5

### `GET /v1/protected/messenger/chat`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** — (WebSocket)
- **Ответ (WebSocket):** обмен сообщениями по WS; не HTTP-ответ
- **Проверка:** WebSocket, HTTP-проверка неприменима

### `POST /v1/protected/messenger/new`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"user_id_list"}
```

- **Коды:** 201, 400, 500
- **Ответ (объект):** данные чата (`id`, `user_list`, `type`)
- **Проверка:** не проверялось (создаёт данные)

### `POST /v1/protected/messenger/new_group`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"avatar", "title", "user_id_list"}
```

- **Коды:** 201, 400, 500
- **Ответ (объект):** список чатов пользователя

### `POST /v1/protected/messenger/edit`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"avatar", "chat_id", "description", "title"}
```

- **Коды:** 200, 400, 500
- **Ответ (пусто):** 200

### `POST /v1/protected/messenger/invitable`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"chat_id"}
```

- **Коды:** 200, 204, 400, 500
- **Ответ (объект):** пользователи, которых можно пригласить

---

## 6. Контент (content_service)

Маршрутов: 17

### `POST /v1/protected/content/posts`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"author_id", "media", "text"}
```

- **Коды:** 200, 500
- **Ответ (объект):** `id` созданного поста

### `POST /v1/protected/content/comments`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"content_id", "content_type", "text"}
```

- **Коды:** 200, 500
- **Ответ (объект):** `{id}` созданного комментария

### `POST /v1/protected/content/likes`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"content_id", "content_type"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `PUT /v1/protected/content/posts/<postid>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `postid` — int
- **Тело запроса:**

```json
{"media", "post_id", "text"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `PUT /v1/protected/content/comments/<commentid>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `commentid` — int
- **Тело запроса:**

```json
{"comment_id", "content_id", "content_type", "text"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `GET /v1/protected/content/posts/<postid>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `postid` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** `id`, `text_content`, `media_content`, `author_id`, `created_at` (string)
- **Проверено на проде:** `500`

```
error: RangeError (length): Invalid value: Valid value range is empty: 0
```

### `GET /v1/protected/content/comments/<commentid>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `commentid` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (объект):** `id`, `text_content`, `author_id`, `content_id`, `content_type`, `created_at`

### `GET /v1/protected/content/posts/author/<author_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `author_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** `id_list`; ключи `0..N`: `{id, text_content, media_content, author_id, created_at, likes, reposts, comments, username, avatar, is_liked}`

### `GET /v1/protected/content/posts/author`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** то же, что `/content/posts/author/<author_id>`
- **Проверено на проде:** `500`

```
error: FormatException: Invalid radix-10 number (at character 1) author
```

### `GET /v1/protected/content/posts/author/null`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** то же

### `GET /v1/protected/content/posts`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** `id_list`; ключи `0..N`: `{id, text_content, media_content, author_id, created_at, likes, reposts, comments, username, avatar, is_liked, is_subscribed}`
- **Проверено на проде:** `500`

```
FormatException: Unexpected character (at character 1) Route not found
```

### `GET /v1/protected/content/comments/<content_type>/<content_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `content_type` — int, `content_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (объект):** ключи `0..N`: `{id, text_content, author_id, content_id, content_type, created_at, likes, username, avatar, is_liked}`

### `GET /v1/protected/content/find/posts/<post_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `post_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 401, 500
- **Ответ (объект):** `users`, `posts`; ключи `0..N` — поля поста + `avatar`, `username`

### `GET /v1/protected/content/find/posts/`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 401, 500
- **Ответ (объект):** то же

### `GET /v1/protected/content/find/clips/<clip_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `clip_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 204, 500
- **Ответ (объект):** поля клипа: `id`, `text_content`, `media_content`, `author_id`, `created_at`, `likes`, `reposts`, `username`, `avatar`, `is_liked`

### `DELETE /v1/protected/content/posts/<post_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `post_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `DELETE /v1/protected/content/comments/<comment_id>`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Параметры пути:** `comment_id` — int
- **Тело запроса:** не требуется
- **Коды:** 200, 500
- **Ответ (пусто):** 200

---

## 7. Помощь (help_service)

Маршрутов: 3

### `POST /v1/help/registration`

- **Авторизация:** нет (публичный)
- **Тело запроса:**

```json
{"mail", "text"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `POST /v1/protected/help/bugs`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"text"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** 200

### `POST /v1/protected/help/report`

- **Авторизация:** Bearer → шлюз подставляет `X-User-Id`
- **Тело запроса:**

```json
{"text"}
```

- **Коды:** 200, 500
- **Ответ (пусто):** 200

---

## 8. Публичные префиксы шлюза

| Путь | Сервис | Авторизация |
|---|---|---|
| `/v1/version` | nginx | нет |
| `/v1/auth/*` | auth_service | нет |
| `/v1/otp/*` | otp_service | нет |
| `/v1/users/signup`, `/login`, `/check`, `/newpass` | user_service | нет |
| `/v1/help/registration` | help_service | нет |
| `/v1/protected/users/*` | user_service | Bearer |
| `/v1/protected/garage/*` | cars_service | Bearer |
| `/v1/protected/messenger/*` | messenger_service | Bearer (кроме `/chat`) |
| `/v1/protected/content/*` | content_service | Bearer |
| `/v1/protected/help/*` | help_service | Bearer |
| `/v1/protected/notification/*` | notification_service | Bearer |
| `/v1/protected/face_taxi/*`, `/meetings/*` | meetings_service | Bearer |
| `/v1/protected/hobbies/*` | hobbies_service | Bearer |
| `/v1/protected/tags` | tags_service | Bearer |
| `/internal/*` | разные | только изнутри кластера |

---

## 9. Сервисы без исходников

На проде для них есть только AOT-бинарники. Маршруты извлечены из строк бинарника; схемы тел и ответов не описаны — нужны исходники или ручной разбор.

| Сервис | Маршруты |
|---|---|
| auth_service | `/v1/auth/token/release`, `/verify`, `/refresh`, `/revoke` |
| otp_service | `/v1/otp`, `/v1/otp/mail`, `/v1/otp/sms`, `/v1/SendSMS` |
| notification_service | `/notification/send`, `/token/create`, `/preferences[/<target_user_id>]`, `/settings[/init…]` |
| meetings_service | `/face_taxi/{create, accept_response/<id>, respond/<id>, cancel_by_user, close_by_driver, get_role, set_role, status/<id>, has_active_trip, list/open, list/all}`, `/meetings/{filters, filters/update, ratings/create, ratings/mutual/<rideId>, user_profiles/*}` |
| hobbies_service | `/hobbies/{name, name/<name>, create, subscribe, join_chat, find/<match_text>}`, `/hobbies/events/*`, `/hobbies/focuses/*`, `/hobbies/users/name/<name>` |
| tags_service | `/tags`, `/tags/<id>`, `/tags/create`, `/tags/search`, `/tags/upsert-internal` |
| kino_service | прикладных маршрутов в бинарнике не найдено |

**Проверено живьём:**

| Запрос | Ответ |
|---|---|
| `POST /v1/otp/mail` (пустое тело) | `400` `{"error":"Отсутствует адрес электронной почты"}` |
| `POST /v1/otp/sms` (пустое тело) | `500` `error: type 'Null' is not a subtype of type 'String'` |
| `GET /v1/protected/face_taxi/get_role` | `200` `{"user_id":212,"status":"passenger","avatar":...,"fullname":...,"username":...,"is_looking_for":true,"is_admin":true}` |
| `GET /v1/protected/face_taxi/list/open` | `403` |
| `GET /v1/protected/meetings/user_profiles/get` | `200` `{"user_id":212,"fullname":...,"username":...,"avatar":...,"description":"","is_looking_for":true}` |
| `GET /v1/notification/settings` | `200` (тело пустое) |
| `GET /v1/protected/hobbies/name` | `200` |
| `GET /v1/protected/hobbies` | `404` |

---

## 10. Найденные дефекты

Все подтверждены вызовом на проде 2026-10-20.

| № | Маршрут | Ответ | Причина в коде |
|---|---|---|---|
| 1 | `GET /v1/protected/users/find/{username}` | `500 Bad state: No element` | `findUser` обращается к первому элементу пустого результата |
| 2 | `GET /v1/protected/users/friends` | `500 Converting object to an encodable object failed` | список объектов `User` кодируется в JSON без маппинга в `Map` |
| 3 | `GET /v1/protected/content/posts` | `500 FormatException` | маршрут не найден внутри сервиса — обработчик не зарегистрирован |
| 4 | `GET /v1/protected/content/posts/author` | `500 FormatException: Invalid radix-10 number` | `author` парсится как int в `<author_id>` |
| 5 | `GET /v1/protected/content/posts/{id}` | `500 RangeError` | пустой результат из БД разыменовывается без проверки |
| 6 | `GET /v1/protected/messenger/chats` | `404 Route not found` | маршрут закомментирован в коде |
| 7 | `GET /v1/protected/hobbies` | `404` | корневой маршрут не объявлен |
| 8 | `POST /v1/otp/sms` (без тела) | `500` вместо `400` | нет проверки на null |

### Прочие расхождения

- **Тип `user_id`:** в `/users` — строка (`"212"`), в `/users/{id}` — число (`212`). В токенах исправлено на число.
- **`total_score`** в `getUserById` вычисляется из поля аватара (`rankedUsers[0][3].toString()`) — вероятно, ошибка индекса. В живом ответе поле отсутствует.
- **Пустой результат:** где-то `204`, где-то `200 {}`.
- **`content_service`:** маршруты `clips`/`stories` закомментированы — в API не работают, хотя обработчики есть.

---

> Изменений на сервисах не вносилось. Документ отражает текущее поведение прода, включая дефекты.
