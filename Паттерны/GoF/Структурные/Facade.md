Как и с мостом - полупустой из-за очевидности паттерн.
Требует, чтобы пользователь мог стучаться только через специальный объект, а не как он сам захочет. 
Говоря проще - требует, чтобы приложение было разбито на слои.

# О чём вообще
## Когда?
Когда приложение посложнее A+B.

## Проблема
Слишком много методов в разных кусках, большая часть которых пользователю не нужна.

## Решение
Обрезаем всё, что не надо (private в помощь), говорим стучаться только вот через этот метод - и готово.

# Список литературы:
https://refactoring.guru/ru/design-patterns/facade