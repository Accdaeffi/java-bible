# Общее
## Когда?
У нас есть микросервисы. Штук 20 разных. Все они хорошо работают, клиенты стучат напрямую к каждому микросервису, а значит - знают их адрес.

## Проблема
Внезапно понадобилось забанить целую группу IP - например, рынок Африки не интересует, а DDOS оттуда заваливается. Или же постоянно дохнут ноды и из-за этого клиенты часто стучат туда, где уже никого нет.
Или просто добавить логирование, откуда какой запрос приходит.

Придётся залезать в каждый микросервис, копировать во все один и тот же код и молиться, что никогда не понадобится что-то менять. Или делать очередной микросервис, который точно так же сможет переезжать, но который сможет сказать, что делать с запросами.

## Решение
А давайте действительно сделаем один микросервис - но не для информирования других, а для обработки самих запросов? Назовём его "Ворота".
Запрос приходит на ворота, там на него смотрят, шлют нафиг, если не красивый, а если красивый - говорят куда идти. Возможно, даже записывают, кто и откуда пришёл.

Весь функционал описанный в "проблеме" добавляется и редактируется за минуты, а сами ворота ещё могут дополнительно служить балансировщиком - если есть несколько инстансов одного сервиса.

## Задачи
- Роутинг
- Безопастность
- Ограничение частоты запросов
- Мониторинг/ логирование
- Сине-зелёный деплой
- Кэширование
- Реализация паттерна "Душитель"

## Плюсы
1. Скрывает реализацию от клиента
2. Безопастнее
3. Удобнее что-то менять в будущем

## Минусы
1. Единая точка падения
2. Надо реализовывать

# Реализации
## Типы
- Hardware
- SAAS (Software as a service)
- Веб-сервисы, которые настроены как прокси
- Направленные на разработчиков (Spring Cloud Gateway)
- Можно смешивать и просто делать что-то другое