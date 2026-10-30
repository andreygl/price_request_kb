---
name: drawio-db-schema
description: Правила формирования схем связей таблиц БД в draw.io - стили таблиц и связей, шрифты, цвета, стрелки, экспорт в PNG. Использовать при создании или правке схемы БД.
---

# Схема связей таблиц БД в draw.io

Правила единого оформления диаграмм связей таблиц базы данных. Эталон стиля - файл `Пример связей БД.drawio` в корне репозитория.

## Что рисуем

- Каждая таблица - овал с названием таблицы. Поля внутри овала НЕ перечисляем.
- Связи - линии между овалами с подписью имени поля связи (FK).
- Схема кладётся в `<Страница>.assets/Схема данных.drawio`, экспорт в `<Страница>.assets/Схема данных.png`.

## Стили

### Таблица (овал)

Стиль ячейки:

```
ellipse;whiteSpace=wrap;html=1;strokeWidth=1;fontFamily=Arial;fontSize=10;align=center;strokeColor=#254569;fontColor=#254569;fillColor=#ffffff;strokeOpacity=100;
```

- Шрифт названия таблицы: Arial, размер 10.
- Цвет обводки и текста: `#254569`.
- Заливка: белая.

### Связь по внешнему ключу

Стиль ребра:

```
edgeStyle=none;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;strokeColor=#366497;fontSize=11;fontColor=#366497;startArrow=classic;startFill=1;endArrow=none;
```

- Цвет линии и подписи: `#366497`.
- Шрифт подписи: Arial, размер 11.
- Стрелка - классическая (закрашенный треугольник) и направлена в сторону родительской таблицы (сторона "один").
- Подпись - имя поля FK, например `tenderid`, `lotid`, `resultid`.

### Связь без внешнего ключа

Та же связь плюс пунктир:

```
edgeStyle=none;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;strokeColor=#366497;fontSize=11;fontColor=#366497;startArrow=classic;startFill=1;endArrow=none;dashed=1;
```

Примеры: `lot.lotnumber` -> `result.lotnumber`, `lotitem.lotitemnumber` -> `resultitem.position_number`.

### Группирующая область (если нужна)

Пунктирный прямоугольник вокруг набора таблиц:

```
rounded=0;whiteSpace=wrap;html=1;strokeWidth=1;fontFamily=Verdana;fontSize=11;align=center;strokeColor=#2E5480;dashed=1;fontColor=#2E5480;verticalAlign=top;dashPattern=1 4;strokeOpacity=100;
```

- Цвет: `#2E5480`, шрифт Verdana, размер 11.

## Цвета и шрифты (сводно)

| Элемент | Цвет | Шрифт | Размер |
| --- | --- | --- | --- |
| Таблица (овал) | `#254569` | Arial | 10 |
| Связь и подпись | `#366497` | Arial | 11 |
| Группирующая область | `#2E5480` | Verdana | 11 |

## Формат файла

- XML с `<mxfile>` -> `<diagram>` -> `<mxGraphModel>` -> `<root>`.
- Узлы-таблицы: `<mxCell ... vertex="1" parent="1">` с `<mxGeometry x y width height as="geometry"/>`.
- Рёбра: `<mxCell ... edge="1" parent="1" source="<id таблицы>" target="<id таблицы>">`.

## Экспорт в PNG

1. Предпочтительно: draw.io desktop CLI
   `draw.io.exe --export --format png --output "Схема данных.png" "Схема данных.drawio"`.
2. Если CLI не принимает `--export` (сборки 16.9.1 и 16.16.0 флаг отклоняют), собрать эквивалентный SVG в тех же стилях и отрендерить headless Chrome:
   `chrome.exe --headless --disable-gpu --hide-scrollbars --screenshot="Схема данных.png" --window-size=<w>,<h> --force-device-scale-factor=2 "Схема данных.svg"`.
3. В ручной SVG-сборке линии обрезаются по границе овала, а не идут от центра к центру: иначе наконечник оказывается под белой заливкой овала и на PNG стрелки не видно. Наконечник - `<marker>` с `refX` на острие, конец линии - с зазором 1-2 px от границы.
4. Вставить PNG в страницу:
   `![Схема связей между таблицами](<Страница>%20assets/Схема%20данных.png)`.

## Чек-лист

- [ ] Каждая таблица - овал с названием, без полей.
- [ ] Связи подписаны именем поля FK.
- [ ] Стрелка указывает на родителя; не-FK связи пунктиром.
- [ ] На PNG у каждой связи виден наконечник, он не спрятан под овалом.
- [ ] Цвета и шрифты соответствуют таблице выше.
- [ ] Файлы `.drawio` и `.png` лежат в `<Страница>.assets/`.
