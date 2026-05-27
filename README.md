# Улицы и адреса Челябинска с координатами

MySQL-дамп адресной базы Челябинска: улицы, дома, координаты и поле сортировки/приоритета. Набор данных актуализирован для моего проекта на момент 2024 года и теперь опубликован в открытый доступ.

Изначально база использовалась в проекте `demo55.gkh-inform74.ru`. Проект не взлетел, поэтому сами данные я решил оставить доступными для тех, кому они могут пригодиться.

## Что внутри

Файл [`streets8.sql`](./streets8.sql) содержит таблицу `streets8`:

| Поле | Тип | Описание |
| --- | --- | --- |
| `id` | `int(11)` | Уникальный идентификатор записи, `AUTO_INCREMENT` |
| `concat` | `varchar(50)` | Полный адрес в текстовом виде: улица/территория и номер дома |
| `geo` | `varchar(25)` | Координаты в формате `широта, долгота` |
| `priority` | `tinyint(1)` | Поле приоритета/сортировки |

В дампе:

- 41 992 адресные записи;
- кодировка `utf8mb4`;
- движок таблицы `InnoDB`;
- первичный ключ по `id`;
- уникальный индекс по `concat`.

Пример записи:

```sql
('1-й Аэропорт, 1-я Увельская улица, 1', '55.291244, 61.512876', 1, 1)
```

## Импорт в MySQL/MariaDB

Создайте базу данных и импортируйте дамп:

```bash
mysql -u USER -p -e "CREATE DATABASE chelyabinsk_streets CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;"
mysql -u USER -p chelyabinsk_streets < streets8.sql
```

Если база уже создана:

```bash
mysql -u USER -p DATABASE_NAME < streets8.sql
```

Дамп был выгружен из MariaDB `10.5.29`, но должен нормально импортироваться в современные версии MySQL и MariaDB.

## Примеры запросов

Найти адрес по части строки:

```sql
SELECT *
FROM streets8
WHERE `concat` LIKE '%Блюхера, 10%';
```

Получить координаты конкретного адреса:

```sql
SELECT geo
FROM streets8
WHERE `concat` = 'улица Блюхера, 10';
```

Разделить координаты на широту и долготу средствами SQL:

```sql
SELECT
  `concat`,
  SUBSTRING_INDEX(geo, ',', 1) AS lat,
  TRIM(SUBSTRING_INDEX(geo, ',', -1)) AS lon
FROM streets8
WHERE `concat` = 'улица Блюхера, 10';
```

## Для чего можно использовать

- автодополнение адресов по Челябинску;
- геокодинг уже известных адресов;
- карты, справочники, внутренние CRM и ЖКХ-сервисы;
- проверка и нормализация пользовательского ввода;
- быстрый локальный справочник без обращения к внешним API.

## Важные замечания

Данные опубликованы как есть. Перед использованием в продакшене проверьте полноту и точность адресов для своего сценария, потому что городская адресная база может меняться.

Поле `geo` хранится строкой, а не отдельными числовыми колонками. Если планируются геопоиск, сортировка по расстоянию или пространственные индексы, лучше при импорте дополнительно разнести координаты в `DECIMAL`/`DOUBLE` или `POINT`.

Координаты полностью пригодны для использования на Яндекс Картах. В других картографических сервисах возможны небольшие неточности или смещения из-за различий в адресных базах, геокодировании и отображении объектов.

## Лицензия

Репозиторий опубликован в открытый доступ. Если вам нужен формально закрепленный режим использования, добавьте файл `LICENSE` с подходящей открытой лицензией.

## Дополнительно

Данные актуальны на момент 2024 года.

### Конвертация в другие форматы

Самый простой путь: сначала импортировать `streets8.sql` в MySQL/MariaDB, а затем выгрузить таблицу в нужный формат.

#### CSV

Выгрузите TSV из MySQL:

```bash
mysql -u USER -p --batch --raw DATABASE_NAME \
  -e 'SELECT id, `concat`, geo, priority FROM streets8' \
  > streets8.tsv
```

Конвертируйте TSV в CSV через стандартный модуль Python:

```bash
python -c "import csv; import sys; csv.writer(open('streets8.csv','w',newline='',encoding='utf-8')).writerows(csv.reader(open('streets8.tsv',encoding='utf-8'), delimiter='\t'))"
```

#### JSON

Через `jq` из TSV-выгрузки:

```bash
mysql -u USER -p --batch --raw --skip-column-names DATABASE_NAME \
  -e 'SELECT id, `concat`, geo, priority FROM streets8' \
  | jq -R -s '
      split("\n")[:-1]
      | map(split("\t"))
      | map({
          id: (.[0] | tonumber),
          address: .[1],
          geo: .[2],
          priority: (.[3] | tonumber)
        })
    ' > streets8.json
```

#### SQLite

Установите `sqlite3`, выгрузите данные в TSV:

```bash
mysql -u USER -p --batch --raw DATABASE_NAME \
  -e 'SELECT id, `concat`, geo, priority FROM streets8' \
  > streets8.tsv
```

Создайте таблицу:

```sql
CREATE TABLE streets8 (
  id INTEGER PRIMARY KEY,
  concat TEXT UNIQUE,
  geo TEXT,
  priority INTEGER
);
```

Импортируйте TSV:

```bash
sqlite3 streets8.sqlite ".mode tabs" ".import --skip 1 streets8.tsv streets8"
```

#### PostgreSQL

Удобный вариант: выгрузить TSV из MySQL и загрузить его через `psql`.

```bash
mysql -u USER -p --batch --raw DATABASE_NAME \
  -e 'SELECT id, `concat`, geo, priority FROM streets8' \
  > streets8.tsv
```

В PostgreSQL:

```sql
CREATE TABLE streets8 (
  id integer PRIMARY KEY,
  concat varchar(50) UNIQUE,
  geo varchar(25),
  priority smallint NOT NULL DEFAULT 1
);
```

Импорт:

```bash
psql -d DATABASE_NAME -c "\copy streets8(id, concat, geo, priority) FROM 'streets8.tsv' WITH (FORMAT csv, DELIMITER E'\t', HEADER true)"
```

После импорта можно добавить отдельные числовые колонки для координат или PostGIS-поле, если нужен геопоиск.
