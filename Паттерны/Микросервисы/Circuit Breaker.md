# О чём вообще
## Когда?
У нас есть сайт зоопарка. Один сервис отвечает за фронт, один - за животных и ещё один - за билеты.
На фронт приходят запросы по животным и билетым, они пересылаются на соответствующие сервисы, обрабатываются, результат возвращается на фронт.

## Проблема
Сервис животных упал, однако на фронт продолжают приходить запросы к нему, которые не могут получить ответа и накапливаются. 
1 запрос - 1 поток, запросов много, а вот потоков ограниченное количество. 
В один прекрасный момент они кончаются и новые просто не создаются - фронт вообще перестаёт реагировать на запросы. Вся система мертва.

## Решение
А давайте будем считать, сколько запросов были неуспешными/не завершилсь как надо.
Если их достаточно много - то, с большой вероятностью, сервис упал и надо как-то об этом сообщать, например, кидать особую ошибку или запускать резервный.

## Аналоги
1. Запустить несколько микросервисов каждого типа. Один упал - другие заменят.
2. Шаблон Bulkhead. Пусть у каждого типа запросов будет свой максимум за раз, например - 10. Первые 10 зависнут и займут свои ресурсы, но вот уже 11 сразу выдаст ошибку.
3. Повесить timeout на запрос. Трудно понять, какой именно timeout вешать, чтобы и не слишком долго, и не фейлилось всё подряд.  

# Общее

## Плюсы
1. Легко в реализации
2. Рабоче
3. Можно приделать кэшированные ответы, что вообще круто и можно скрыть, что сервис упал

# Реализация

## По человечески
Создаём специальный прокси, через которые будут проходить все запросы. 
Это прокси должно иметь 3 состояния: 
1. Closed (все запросы сразу идут куда надо, начальное состояние).
2. Open (все запросы сразу валятся).
3. Half-open (запросы идут когда надо, но количество падений подсчитывается).

При превышении числа ошибок в closed - переход в open, оттуда, по таймеру - в half-open. Если по результатам анализа в half-open всё ещё слишком много падений - возврат в open, иначе - в closed. 

## Spring
Максимально удобно делать в Spring Cloud - достаточно лишь импортировать spring-cloud-starter-circuitbreaker{-reactor}-resilience4j и похимичить в SpringCloudGateway, а именно прописать в Spring Router у нужного пути:
```java
.route(r -> r.path("/api/v1/beer/*/inventory")
    .filters(f -> f.circuitBreaker(c -> c.setName("inventoryCB")
				.setFallbackUri("forward:/inventory-failover")
				.setRouteId("inv-failover")
			))
    .uri("lb://inventory-service")
    .id("inventory-service"))
```
и тут же прописать сам путь до сервиса с аварийной функциональностью.

# Список литературы:
https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker
https://sysout.ru/otkazoustojchivost-mikroservisov-shablon-circuit-breaker/
