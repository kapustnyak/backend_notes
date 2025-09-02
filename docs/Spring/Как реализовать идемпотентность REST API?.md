**Идемпотентность** — это свойство операции давать один и тот же эффект (и, желательно, тот же ответ), если выполнить её один или много раз подряд. В REST это критично для повторов при сетевых сбоях, ретраях клиентских библиотек и балансировщиков.

Ниже — практические приёмы для Java-сервисов (без привязки к конкретному фреймворку; примеры — на JAX-RS и стандартных возможностях Java).

#### 1) Дизайн ресурсов под идемпотентность

**Методы по смыслу:**
- GET, HEAD, OPTIONS — не должны менять состояние (safe).
- PUT и DELETE — по определению **идемпотентны**: повторный вызов приводит к тому же состоянию ресурса.
- POST — **не идемпотентен** по умолчанию (создание «каждый раз нового»). Его делают идемпотентным с ключами.

**Паттерны:**
- **PUT с клиентским ID** (upsert): клиент выбирает id, сервер создает или обновляет один и тот же ресурс.
- **DELETE «мягкий»**: повторный DELETE возвращает 204 No Content (или 200 с тем же представлением), даже если ресурс уже удалён/отсутствует.

**Мини-пример (JAX-RS):**
```java
@Path("/users")
public class UserResource {

    private final UserRepo repo = new InMemoryUserRepo();

    @PUT @Path("/{id}") // upsert: повторный вызов задаёт то же состояние
    @Produces("application/json") @Consumes("application/json")
    public Response upsert(@PathParam("id") String id, UserDto dto) {
        User u = repo.saveOrUpdate(id, dto); // атомарно в БД/хранилище
        return Response.ok(u).build();
    }

    @DELETE @Path("/{id}") // повтор — не ошибка
    public Response delete(@PathParam("id") String id) {
        repo.deleteIfExists(id);
        return Response.noContent().build(); // 204 всегда
    }
}
```

#### 2) Идемпотентность POST через Idempotency-Key

Когда нужен именно POST (например, платеж, заказ), добавляйте **идемпотентный ключ** (заголовок Idempotency-Key или параметр). Сервер:
1. принимает ключ,
2. **единожды** выполняет операцию,
3. кэширует **итоговый ответ** и связывает его с ключом,
4. при повторе с тем же ключом мгновенно возвращает тот же ответ.

Мини-реализация обработчика ключа (фильтр):
```java
@Provider
public class IdempotencyFilter implements ContainerRequestFilter {
    private final IdempotencyStore store = new InMemoryIdemStore();

    @Override
    public void filter(ContainerRequestContext ctx) {
        String key = ctx.getHeaderString("Idempotency-Key");
        if (key == null || key.isBlank()) return; // не применяем

        Optional<CachedResponse> cached = store.find(key);
        if (cached.isPresent()) {
            // короткое замыкание: вернуть тот же ответ
            Response resp = Response.status(cached.get().status())
                                    .entity(cached.get().body())
                                    .build();
            ctx.abortWith(resp);
        } else {
            // пометим запрос как "в работе" (для дедупа гонок)
            if (!store.tryStart(key)) {
                ctx.abortWith(Response.status(409).entity("Duplicate in progress").build());
            }
        }
    }
}
```

Сохранение результата в конце хендлера:
```java
public Response createOrder(RequestDto dto, @HeaderParam("Idempotency-Key") String key) {
    Order result = service.createOnce(dto); // атомарно внутри транзакции
    Response resp = Response.status(201).entity(result).build();
    if (key != null && !key.isBlank()) {
        idemStore.save(key, 201, result, /* ttl */ Duration.ofHours(24));
    }
    return resp;
}
```

#### 3) Защита на уровне БД (exactly-once эффект)

Идемпотентность невозможна без **уникальных ограничений** и транзакций:
- Установите UNIQUE на бизнес-идентификатор (например, order_id или client_idempotency_key).
- Вставка с повтором должна приводить не к дублю, а к «получить существующий результат».
- Оборачивайте в **транзакцию**: проверка → запись → фиксация. На конфликте — читать существующее и возвращать тот же ответ.
```java
public Order createOnce(RequestDto dto) {
    return tx(() -> {
        Optional<Order> existing = orderRepo.findByIdemKey(dto.idemKey());
        if (existing.isPresent()) return existing.get();

        Order created = orderRepo.insert(dto); // может бросить из-за UNIQUE
        return created;
    });
}
```

#### 4) Ретраи и «дважды отправлено, но неясно, выполнилось ли»
- Клиент может повторить запрос из-за таймаута ответа. Благодаря ключу сервер вернёт **тот же** результат (или 202/409 по вашей политике).
- Логируйте и возвращайте стабильные коды:
    - 201 Created при первом POST с Location.
    - Повтор — 200 OK или 201 с тем же Location/телом (оба допустимы, важна **стабильность**).
- Для PUT/DELETE ответ неизменен при повторах: 200/204.

#### 5) Конкурентность и гонки
- В IdempotencyStore нужна **атомарная резервация ключа** (tryStart) и TTL на «висящие» операции.
- На уровне БД используйте **уникальные индексы** + **transaction isolation** (обычно READ COMMITTED достаточно, для спорных мест — SERIALIZABLE или оптимистичные версии).
- Если вы публикуете события (Kafka, очередь) — применяйте **outbox-паттерн** с уникальным ключом сообщения, чтобы консюмеры могли делать дедупликацию.

#### 6) Что тестировать
- Повтор одного и того же PUT/DELETE → идентичное состояние и приемлемый одинаковый ответ.
- Два параллельных POST с одним Idempotency-Key → одна запись и одинаковые ответы.
- Таймаут ответа клиента + повтор с тем же ключом → тот же результат.
- Истечение TTL в кэше ключа не нарушает уникальность: при повторе после TTL ответ извлекается из БД.