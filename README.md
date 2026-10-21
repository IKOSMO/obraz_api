API-документация Obraz — PROD
Источник: прод-сервер `185.93.111.249`, снято 2026-10-20. Маршруты, обработчики, поля запроса и коды взяты из активных исходников; схемы ответов — из функций БД; часть маршрутов дополнительно вызвана на живом проде.
Содержание
Общие правила
Пользователи (user_service)
Гараж (cars_service)
Мессенджер (messenger_service)
Контент (content_service)
Помощь (help_service)
Публичные префиксы шлюза
Сервисы без исходников
Найденные дефекты
---
1. Общие правила
Аутентификация
Защищённые маршруты живут под `/v1/protected/...`. Схема:
```
клиент → Authorization: Bearer <accessToken> → nginx-gateway
                                     ↓ auth\_request /auth\_verify
                          auth\_service /v1/auth/token/verify
                                     ↓ 200 + X-User-Id
           nginx подставляет X-User-Id → сервис
```
Клиент отправляет только `Authorization: Bearer <token>`. Заголовок `X-User-Id` выставляет шлюз.
Если отправить `X-User-Id` вручную — шлюз его перезапишет своим значением.
Без токена / с неверным: `401` `{"message":"Отсутствует токен или неверный формат"}`.
Внутренние вызовы между сервисами идут через `/internal/...` — снаружи недоступны.
Формат запросов
	
Тело	JSON, `Content-Type: application/json`
Исключение	`POST /users/avatar` — сырые байты изображения
Исключение	`GET /messenger/chat` — WebSocket
Пути	`<user\_id>`, `<car\_id>` и др. — целые числа
Тело читается как `Map` без типизации	поля не валидируются схемой
Формат ответов
Единого формата ошибок нет. Три варианта:
Формат	Пример	Доля
Текст исключения	`error: Bad state: No element`	большинство `500`
JSON `{"message": ...}`	`{"message":"Невозможно..."}`	часть `400`/`401`
Без тела	`204`, `400`	валидация, пустые списки
Следствие для клиента: на `500` приходит внутренний текст Dart с типами и деталями БД. Стабильно разбирать такие ответы нельзя — обрабатывайте по коду статуса.
Коды
Код	Когда
200	Успех
201	Создано
204	Успех, результат пуст
400	Валидация / нет поля
401	Нет токена / неверные данные
403	Нет прав
500	Внутренняя ошибка
Списки приходят объектами, не массивами
Коллекции отдаются как `{"0": {...}, "1": {...}}` — не JSON-массив. Порядок ключей не гарантирован.
---
2. Пользователи (user_service)
Маршрутов: 26
`POST /v1/users/signup`
Авторизация: нет (публичный)
Тело запроса:
```json
{"bdate", "description", "device\_token", "email", "fullname", "license\_plate", "license\_plate\_visibility", "password", "phone", "username"}
```
Коды: 500
Ответ (объект): accessToken, refreshToken, user_id (int)
`POST /v1/users/login`
Авторизация: нет (публичный)
Тело запроса:
```json
{"device\_token", "email", "password", "phone", "username"}
```
Коды: 500
Ответ (объект): accessToken, refreshToken, user_id (int)
`POST /v1/users/check`
Авторизация: нет (публичный)
Тело запроса:
```json
{"email", "number", "username"}
```
Коды: 200, 400, 500
Ответ (объект): {phone} или {email}
`POST /v1/users/newpass`
Авторизация: нет (публичный)
Тело запроса:
```json
{"email", "number", "password", "username"}
```
Коды: 200, 400, 500
Ответ (пусто): тела нет
`POST /v1/protected/users`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"id\_list"}
```
Коды: 204, 500
Ответ (объект): ключ = id: {username, avatar}
`POST /v1/protected/users/`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"id\_list"}
```
Коды: 204, 500
Ответ (объект): то же, что POST /users
`POST /v1/protected/users/sub`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 204, 500
Ответ (объект): ключи 1..N: {id, username, avatar}
`POST /v1/protected/users/change`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: читается как JSON, поля в коде не выделены
Коды: 200, 400, 500
Ответ (пусто): тела нет
`POST /v1/protected/users/avatar`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: читается как JSON, поля в коде не выделены
Коды: 200, 500
Ответ (пусто): тела нет (запрос — сырые байты изображения)
`GET /v1/protected/users`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 500
Ответ (объект): user_id (string), username, fullname, description, sex, avatar (string|null), rating (string), subscribers (string), subscribes (string)
Проверено на проде: `200`
```json
{"user\_id":"212","username":"shtabels","fullname":"Сергей Штабс","description":"Developer","sex":"1","avatar":"https://...","rating":"19","subscribers":"6","subscribes":"13"}
```
`POST /v1/protected/users/subscribe`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"subscribe\_to\_id"}
```
Коды: 201, 500
Ответ (пусто): тела нет, код 201
`GET /v1/protected/users/subscriptions`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 1..N: {user_id, username, fullname, avatar}
Проверено на проде: `200`
```json
{"1":{"user\_id":115,"username":"katya\_","fullname":"Екатерина","avatar":"https://..."}}
```
`GET /v1/protected/users/subscribers`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 1..N: {user_id, username, fullname, avatar}
`GET /v1/protected/users/friends`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 1..N: {id, username, avatar}
Проверено на проде: `500`
```json
error: Converting object to an encodable object failed
```
`GET /v1/protected/users/rating`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 1..N: {user_id, username, fullname, avatar, likes, comments, subs, total_score, rating, you (bool)}
Проверено на проде: `200`
```json
{"1":{"user\_id":312,"username":"Liliya\_Chernykh","fullname":"Лилия Черных","avatar":"https://...",...}}
```
`POST /v1/protected/users/settings`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"license\_plate\_visibility"}
```
Коды: 200, 500
Ответ (пусто): тела нет
`GET /v1/protected/users/settings`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 500
Ответ (объект): user_id (int), license_plate_visibility (int)
Проверено на проде: `200`
```json
{"user\_id":212,"license\_plate\_visibility":0}
```
`POST /v1/protected/users/get\_settings`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"id\_list"}
```
Коды: 200, 500
Ответ (объект): ключ = user_id: {license_plate_visibility (int)}
`GET /v1/protected/users/<user\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `user\_id` — int
Тело запроса: не требуется
Коды: 200, 500
Ответ (объект): id (int), username, fullname, avatar (string|null), description, subscribers (string), subscribes (string), rating (string), subscribed (bool)
Проверено на проде: `200`
```json
{"id":212,"username":"shtabels","fullname":"Сергей Штабс","avatar":"https://...","description":"Developer","subscribers":"6","subscribes":"13","rating":"19","subscribed":false}
```
`GET /v1/protected/users/friends/<user\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `user\_id` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 1..N: {id, username, avatar}
`GET /v1/protected/users/subscribers/<user\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `user\_id` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 1..N: {user_id, username, fullname, avatar}
`GET /v1/protected/users/subscriptions/<user\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `user\_id` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 1..N: {user_id, username, fullname, avatar}
`GET /v1/protected/users/find/`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): то же, что /users/find/<username>, но список всех
`GET /v1/protected/users/find/<username>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `username` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 1..N: {id, username, avatar, description, has_stories (bool), common_friends}
Проверено на проде: `500`
```json
error: Bad state: No element
```
`GET /v1/protected/users/license\_plate\_visibility`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): id_list: список user_id с видимостью номера
`POST /v1/protected/users/find/license\_plate`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"id\_list", "license\_plate"}
```
Коды: 200, 204, 500
Ответ (объект): id_list, users; либо 204
3. Гараж (cars_service)
Маршрутов: 15
`POST /v1/protected/garage/vin/check`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"vin"}
```
Коды: 200, 204, 500
Ответ (пусто|объект): 200 если VIN свободен, 204 если занят
`GET /v1/protected/garage/makes`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 500
Ответ (объект): ключи 0..N: {id, name, country}
Проверено на проде: `200`
```json
{"0":{"id":1,"name":"Abarth","country":"Италия"},"1":{"id":2,"name":"AC","country":"Великобритания"}}
```
`GET /v1/protected/garage/models/<make\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `make\_id` — int
Тело запроса: не требуется
Коды: 200, 500
Ответ (объект): ключи 0..N: {id, name, year_from, year_to, make_id}
`POST /v1/protected/garage/body\_types`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 500
Ответ (объект): ключи 0..N: {id, name}
`POST /v1/protected/garage/modifications`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"body\_type\_id", "make\_id", "model\_id", "year"}
```
Коды: 200, 204, 500
Ответ (объект): ключи 0..N: {id, make_id, model_id, body_type_id, drive_type, fuel_type, is_hybrid, transmission, engine_capacity, power}
`POST /v1/protected/garage`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"body\_type\_id", "cng", "color", "ctc", "description", "drive\_type", "engine\_capacity", "fuel\_type", "hybrid", "license\_plate", "make\_id", "media", "media\_ctc", "mileage", "model\_id", "modification\_id", "no\_mileage\_in\_russia", "power", "steering\_wheel", "transmission", "vin", "year"}
```
Коды: 201, 403, 500
Ответ (пусто): 201 при успехе, 403 если нет прав
`GET /v1/protected/garage/<user\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `user\_id` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): ключи 0..N: {id, vin, ctc_number, license_plate, make_id, make_name, model_id, model_name, steering_wheel, year_of_release, body_id, body_name, drive_type, fuel_type, hybrid, transmission, engine_capacity, power, cng, mileage, no_mileage_in_russia, color, description, media, user_id, status}
Проверено на проде: `200`
```json
{"0":{"id":300,"vin":null,"ctc\_number":null,"license\_plate":"A324AA.324","make\_id":2,"make\_name":"AC",...}}
```
`POST /v1/protected/garage/car/edit`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"body\_type\_id", "cng", "color", "ctc", "description", "drive\_type", "engine\_capacity", "fuel\_type", "hybrid", "id", "license\_plate", "make\_id", "media", "media\_ctc", "media\_links\_list", "mileage", "model\_id", "modification\_id", "no\_mileage\_in\_russia", "power", "steering\_wheel", "transmission", "vin", "year"}
```
Коды: 200, 500
Ответ (пусто): 200
`DELETE /v1/protected/garage/car/<car\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `car\_id` — int
Тело запроса: не требуется
Коды: 200, 500
Ответ (пусто): 200
`POST /v1/protected/garage/car/confirm`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"ctc\_number", "id", "license\_plate", "media\_ctc", "vin"}
```
Коды: 200, 500
Ответ (пусто): 200
`GET /v1/protected/garage/find/`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): то же, что /garage/find/<match_text>
`GET /v1/protected/garage/find/<match\_text>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `match\_text` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): id_list, users; ключи 0..N — как у /garage/<user_id> + username, avatar, user_id, status
Проверено на проде: `200`
```json
{"0":{"id":207,"vin":null,"license\_plate":"T685OO.25","make\_id":344,"make\_name":"Toyota",...}}
```
`GET /v1/protected/garage/find/by\_license\_plate/`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): то же
`GET /v1/protected/garage/find/by\_license\_plate/<license\_plate>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `license\_plate` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): cars, users, id_list + поля машины
`GET /v1/protected/garage/check/<license\_plate>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `license\_plate` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): id_list пользователей с таким номером, либо 204
4. Мессенджер (messenger_service)
Маршрутов: 5
`GET /v1/protected/messenger/chat`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: — (WebSocket)
Ответ (WebSocket): обмен сообщениями по WS; не HTTP-ответ
Проверка: WebSocket, HTTP-проверка неприменима
`POST /v1/protected/messenger/new`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"user\_id\_list"}
```
Коды: 201, 400, 500
Ответ (объект): данные чата (id, user_list, type)
Проверка: не проверялось (создаёт данные)
`POST /v1/protected/messenger/new\_group`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"avatar", "title", "user\_id\_list"}
```
Коды: 201, 400, 500
Ответ (объект): список чатов пользователя
`POST /v1/protected/messenger/edit`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"avatar", "chat\_id", "description", "title"}
```
Коды: 200, 400, 500
Ответ (пусто): 200
`POST /v1/protected/messenger/invitable`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"chat\_id"}
```
Коды: 200, 204, 400, 500
Ответ (объект): пользователи, которых можно пригласить
5. Контент (content_service)
Маршрутов: 17
`POST /v1/protected/content/posts`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"author\_id", "media", "text"}
```
Коды: 200, 500
Ответ (объект): id созданного поста
`POST /v1/protected/content/comments`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"content\_id", "content\_type", "text"}
```
Коды: 200, 500
Ответ (объект): {id} созданного комментария
`POST /v1/protected/content/likes`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"content\_id", "content\_type"}
```
Коды: 200, 500
Ответ (пусто): 200
`PUT /v1/protected/content/posts/<postid>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `postid` — int
Тело запроса:
```json
{"media", "post\_id", "text"}
```
Коды: 200, 500
Ответ (пусто): 200
`PUT /v1/protected/content/comments/<commentid>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `commentid` — int
Тело запроса:
```json
{"comment\_id", "content\_id", "content\_type", "text"}
```
Коды: 200, 500
Ответ (пусто): 200
`GET /v1/protected/content/posts/<postid>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `postid` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): id, text_content, media_content, author_id, created_at (string)
Проверено на проде: `500`
```json
error: RangeError (length): Invalid value: Valid value range is empty: 0
```
`GET /v1/protected/content/comments/<commentid>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `commentid` — int
Тело запроса: не требуется
Коды: 200, 500
Ответ (объект): id, text_content, author_id, content_id, content_type, created_at
`GET /v1/protected/content/posts/author/<author\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `author\_id` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): id_list; ключи 0..N: {id, text_content, media_content, author_id, created_at, likes, reposts, comments, username, avatar, is_liked}
`GET /v1/protected/content/posts/author`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): то же, что /content/posts/author/<author_id>
Проверено на проде: `500`
```json
error: FormatException: Invalid radix-10 number (at character 1) author
```
`GET /v1/protected/content/posts/author/null`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): то же
`GET /v1/protected/content/posts`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): id_list; ключи 0..N: {id, text_content, media_content, author_id, created_at, likes, reposts, comments, username, avatar, is_liked, is_subscribed}
Проверено на проде: `500`
```json
FormatException: Unexpected character (at character 1) Route not found
```
`GET /v1/protected/content/comments/<content\_type>/<content\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `content\_type` — int, `content\_id` — int
Тело запроса: не требуется
Коды: 200, 500
Ответ (объект): ключи 0..N: {id, text_content, author_id, content_id, content_type, created_at, likes, username, avatar, is_liked}
`GET /v1/protected/content/find/posts/<post\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `post\_id` — int
Тело запроса: не требуется
Коды: 200, 204, 401, 500
Ответ (объект): users, posts; ключи 0..N — поля поста + avatar, username
`GET /v1/protected/content/find/posts/`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса: не требуется
Коды: 200, 204, 401, 500
Ответ (объект): то же
`GET /v1/protected/content/find/clips/<clip\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `clip\_id` — int
Тело запроса: не требуется
Коды: 200, 204, 500
Ответ (объект): поля клипа: id, text_content, media_content, author_id, created_at, likes, reposts, username, avatar, is_liked
`DELETE /v1/protected/content/posts/<post\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `post\_id` — int
Тело запроса: не требуется
Коды: 200, 500
Ответ (пусто): 200
`DELETE /v1/protected/content/comments/<comment\_id>`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Параметры пути: `comment\_id` — int
Тело запроса: не требуется
Коды: 200, 500
Ответ (пусто): 200
6. Помощь (help_service)
Маршрутов: 3
`POST /v1/help/registration`
Авторизация: нет (публичный)
Тело запроса:
```json
{"mail", "text"}
```
Коды: 200, 500
Ответ (пусто): 200
`POST /v1/protected/help/bugs`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"text"}
```
Коды: 200, 500
Ответ (пусто): 200
`POST /v1/protected/help/report`
Авторизация: Bearer → шлюз подставляет `X-User-Id`
Тело запроса:
```json
{"text"}
```
Коды: 200, 500
Ответ (пусто): 200
---
7. Публичные префиксы шлюза
Путь	Сервис	Авторизация
`/v1/version`	nginx	нет
`/v1/auth/\*`	auth_service	нет
`/v1/otp/\*`	otp_service	нет
`/v1/users/signup`, `/login`, `/check`, `/newpass`	user_service	нет
`/v1/help/registration`	help_service	нет
`/v1/protected/users/\*`	user_service	Bearer
`/v1/protected/garage/\*`	cars_service	Bearer
`/v1/protected/messenger/\*`	messenger_service	Bearer (кроме `/chat`)
`/v1/protected/content/\*`	content_service	Bearer
`/v1/protected/help/\*`	help_service	Bearer
`/v1/protected/notification/\*`	notification_service	Bearer
`/v1/protected/face\_taxi/\*`, `/meetings/\*`	meetings_service	Bearer
`/v1/protected/hobbies/\*`	hobbies_service	Bearer
`/v1/protected/tags`	tags_service	Bearer
`/internal/\*`	разные	только изнутри кластера
8. Сервисы без исходников
На проде для них есть только AOT-бинарники. Маршруты извлечены из строк бинарника; схемы тел и ответов не описаны — нужны исходники или ручной разбор.
Сервис	Маршруты
auth_service	`/v1/auth/token/release`, `/verify`, `/refresh`, `/revoke`
otp_service	`/v1/otp`, `/v1/otp/mail`, `/v1/otp/sms`, `/v1/SendSMS`
notification_service	`/notification/send`, `/token/create`, `/preferences\[/<target\_user\_id>]`, `/settings[/init
meetings_service	`/face\_taxi/{create,accept\_response/<id>,respond/<id>,cancel\_by\_user,close\_by\_driver,get\_role,set\_role,status/<id>,has\_active\_trip,list/open,list/all}`, `/meetings/{filters,filters/update,ratings/create,ratings/mutual/<rideId>,user\_profiles/\*}`
hobbies_service	`/hobbies/{name,name/<name>,create,subscribe,join\_chat,find/<match\_text>}` , `/hobbies/events/\*`, `/hobbies/focuses/\*`, `/hobbies/users/name/<name>`
tags_service	`/tags`, `/tags/<id>`, `/tags/create`, `/tags/search`, `/tags/upsert-internal`
kino_service	прикладных маршрутов в бинарнике не найдено
Проверено живьём:
Запрос	Ответ
`POST /v1/otp/mail` (пустое тело)	`400` `{"error":"Отсутствует адрес электронной почты"}`
`POST /v1/otp/sms` (пустое тело)	`500` `error: type 'Null' is not a subtype of type 'String'`
`GET /v1/protected/face\_taxi/get\_role`	`200` `{"user\_id":212,"status":"passenger","avatar":...,"fullname":...,"username":...,"is\_looking\_for":true,"is\_admin":true}`
`GET /v1/protected/face\_taxi/list/open`	`403`
`GET /v1/protected/meetings/user\_profiles/get`	`200` `{"user\_id":212,"fullname":...,"username":...,"avatar":...,"description":"","is\_looking\_for":true}`
`GET /v1/notification/settings`	`200` (тело пустое)
`GET /v1/protected/hobbies/name`	`200`
`GET /v1/protected/hobbies`	`404`
9. Найденные дефекты
Все подтверждены вызовом на проде 2026-10-20.
№	Маршрут	Ответ	Причина в коде
1	`GET /v1/protected/users/find/{username}`	`500 Bad state: No element`	`findUser` обращается к первому элементу пустого результата
2	`GET /v1/protected/users/friends`	`500 Converting object to an encodable object failed`	список объектов `User` кодируется в JSON без маппинга в `Map`
3	`GET /v1/protected/content/posts`	`500 FormatException`	маршрут не найден внутри сервиса — обработчик не зарегистрирован
4	`GET /v1/protected/content/posts/author`	`500 FormatException: Invalid radix-10 number`	`author` парсится как int в `<author\_id>`
5	`GET /v1/protected/content/posts/{id}`	`500 RangeError`	пустой результат из БД разыменовывается без проверки
6	`GET /v1/protected/messenger/chats`	`404 Route not found`	маршрут закомментирован в коде
7	`GET /v1/protected/hobbies`	`404`	корневой маршрут не объявлен
8	`POST /v1/otp/sms` (без тела)	`500` вместо `400`	нет проверки на null
Прочие расхождения
Тип `user\_id`: в `/users` — строка (`"212"`), в `/users/{id}` — число (`212`). В токенах исправлено на число.
`total\_score` в `getUserById` вычисляется из поля аватара (`rankedUsers\[0]\[3].toString()`) — вероятно, ошибка индекса. В живом ответе поле отсутствует.
Пустой результат: где-то `204`, где-то `200 {}`.
`content\_service`: маршруты `clips`/`stories` закомментированы — в API не работают, хотя обработчики есть.
---
Изменений на сервисах не вносилось. Документ отражает текущее поведение прода, включая дефекты.
