# О чём вообще
## Когда?
Есть некая древовидная структура приложения - например, заказ в интернет магазине. Мы его красиво упаковали в коробочки и теперь надо понять - а сколько он суммарно весить должен?

## Проблема
Как взвешивать-то? Как вообще отразить, что где как упакованно?

## Решение
Сделаем дерево!
Создаём интерфейс "Orderable", вешаем на коробки и предметы. Коробки могут содержать другие Orderable, предметы - уже нет. Создаём красивую древовидную структуру.
Если надо что-то подсчитать для всего заказа - достаточно лишь добавить метод подсчёте в интерфейс и продумать различную реализацию для коробок и предметов, чтобы коробки ещё считали вес содержимого Orderable.

# Общее

## Применимость
Когда объекты образуют древовидную структуру.

## Плюсы
1. Сильно упрощает работу с древовидными структурами.
2. Упрощает добавление новых классов - достаточно унаследовать их от Orderable.

## Минусы
1. Дополнительные классы.

# Список литературы:
https://refactoring.guru/ru/design-patterns/composite