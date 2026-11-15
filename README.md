# Formula 1 DBMS (`f1db`)

Реляционная in-memory СУБД на Java с поддержкой SQL-подмножества (DDL/DML), индексацией на основе красно-чёрного дерева (`TreeMap`), контролем ссылочной целостности (`PK`/`FK`), отказоустойчивой сериализацией на диск (`.tmp` / `.bak`) и графическим интерфейсом на базе Java Swing.

---

## 1. Архитектура системы

Система спроектирована с разделением на 5 пакетов:
* `f1db.model` — доменная модель БД (`Database`, `Table`, `Column`, `Row`, `DataType`, `TableRowIterator`).
* `f1db.index` — подсистема индексирования (`TableIndex` на базе `TreeMap`).
* `f1db.engine` — парсер и исполнитель SQL-запросов (`SqlEngine`, `Condition`, `QueryResult`).
* `f1db.storage` — слой персистентности с транзакционной подменой файлов (`StorageRepository`).
* `f1db.ui` и `f1db` — графическая консоль управления (`MainFrame`) и точка входа (`F1DatabaseApp`).

### 1.1. Диаграмма классов (Class Diagram)

```mermaid
classDiagram
    direction TB

    class F1DatabaseApp {
        +main(args : String[]) void$
        -initializeDefaultF1Data(engine : SqlEngine) void$
    }

    class Database {
        -serialVersionUID : long$
        -tables : Map&lt;String, Table&gt;
        +getTables() Map&lt;String, Table&gt;
        +getTable(name : String) Table
    }

    class Table {
        -serialVersionUID : long$
        -name : String
        -columns : Map&lt;String, Column&gt;
        -primaryKeyColumn : String
        -rowsByPk : Map&lt;Comparable&lt;?&gt;, Row&gt;
        -indexesByColumn : Map&lt;String, TableIndex&gt;
        +Table(name : String, columnList : List&lt;Column&gt;)
        +getName() String
        +getColumns() Map&lt;String, Column&gt;
        +getPrimaryKeyColumn() String
        +getRowsByPk() Map&lt;Comparable&lt;?&gt;, Row&gt;
        +getIndexesByColumn() Map&lt;String, TableIndex&gt;
        +createIndex(indexName : String, columnName : String) void
        +insertRow(row : Row) void
        +updateRowValue(pkVal : Comparable&lt;?&gt;, columnName : String, newValue : Comparable&lt;?&gt;) void
        +deleteRow(pkVal : Comparable&lt;?&gt;) void
        +iterator() Iterator&lt;Row&gt;
    }

    class Column {
        -serialVersionUID : long$
        -name : String
        -type : DataType
        -primaryKey : boolean
        -foreignKeyTable : String
        -foreignKeyColumn : String
        +Column(name : String, type : DataType, primaryKey : boolean, foreignKeyTable : String, foreignKeyColumn : String)
        +getName() String
        +getType() DataType
        +isPrimaryKey() boolean
        +getForeignKeyTable() String
        +getForeignKeyColumn() String
    }

    class DataType {
        <<enumeration>>
        INT
        FLOAT
        STRING
        DATE
        BOOL
        +parseValue(rawValue : String) Comparable&lt;?&gt;
    }

    class Row {
        -serialVersionUID : long$
        -values : Map&lt;String, Comparable&lt;?&gt;&gt;
        +setValue(columnName : String, value : Comparable&lt;?&gt;) void
        +getValue(columnName : String) Comparable&lt;?&gt;
        +getValues() Map&lt;String, Comparable&lt;?&gt;&gt;
    }

    class TableRowIterator {
        -snapshot : List&lt;Row&gt;
        -currentIndex : int
        +TableRowIterator(rows : Collection&lt;Row&gt;)
        +hasNext() boolean
        +next() Row
    }

    class TableIndex {
        -serialVersionUID : long$
        -indexName : String
        -columnName : String
        -tree : TreeMap&lt;Comparable, Set&lt;Comparable&lt;?&gt;&gt;&gt;
        +TableIndex(indexName : String, columnName : String)
        +getIndexName() String
        +getColumnName() String
        +put(columnValue : Comparable&lt;?&gt;, primaryKey : Comparable&lt;?&gt;) void
        +remove(columnValue : Comparable&lt;?&gt;, primaryKey : Comparable&lt;?&gt;) void
        +search(operator : String, target : Comparable&lt;?&gt;) Set&lt;Comparable&lt;?&gt;&gt;
    }

    class SqlEngine {
        -db : Database
        -repository : StorageRepository
        -autoSave : boolean
        +SqlEngine(db : Database, repository : StorageRepository)
        +getDatabase() Database
        +setAutoSave(autoSave : boolean) void
        +persistIfNeeded() void
        +executeForGui(rawSql : String) QueryResult
        +execute(rawSql : String) String
        -handleCreateTable(sql : String) String
        -handleDropTable(sql : String) String
        -handleCreateIndex(sql : String) String
        -handleInsert(sql : String) String
        -handleSelectStructured(sql : String) QueryResult
        -handleSelect(sql : String) String
        -handleUpdate(sql : String) String
        -handleDelete(sql : String) String
        -ensureNoChildReferences(parentTableName : String, parentPkVal : Comparable&lt;?&gt;) void
        -parseCondition(table : Table, whereClause : String) Condition
    }

    class Condition {
        -columnName : String
        -operator : String
        -targetValue : Comparable&lt;?&gt;
        +Condition(columnName : String, operator : String, targetValue : Comparable&lt;?&gt;)
        +getColumnName() String
        +getOperator() String
        +getTargetValue() Comparable&lt;?&gt;
        +matches(row : Row) boolean
    }

    class QueryResult {
        -selectQuery : boolean
        -columns : List&lt;String&gt;
        -rows : List&lt;Row&gt;
        -statusMessage : String
        +QueryResult(selectQuery : boolean, columns : List&lt;String&gt;, rows : List&lt;Row&gt;, statusMessage : String)
        +ofMessage(message : String) QueryResult$
        +ofSelect(columns : List&lt;String&gt;, rows : List&lt;Row&gt;, statusMessage : String) QueryResult$
        +isSelectQuery() boolean
        +getColumns() List&lt;String&gt;
        +getRows() List&lt;Row&gt;
        +getStatusMessage() String
    }

    class StorageRepository {
        -dbFilePath : Path
        -tmpFilePath : Path
        -bakFilePath : Path
        +StorageRepository(fileName : String)
        +save(db : Database) void
        +load() Database
    }

    class MainFrame {
        -engine : SqlEngine
        -repository : StorageRepository
        -tableList : JList&lt;String&gt;
        -sqlEditor : JTextArea
        -resultTable : JTable
        +MainFrame(engine : SqlEngine, repository : StorageRepository)
        -executeSqlQuery(rawSql : String, refreshTablesOnMutate : boolean) void
        -renderTable(columns : List&lt;String&gt;, rows : List&lt;Row&gt;) void
        -refreshSidebar(preserveSelection : String) void
    }

    F1DatabaseApp ..> StorageRepository : uses
    F1DatabaseApp ..> SqlEngine : creates
    F1DatabaseApp ..> MainFrame : launches
    MainFrame --> SqlEngine : uses
    MainFrame --> StorageRepository : uses
    MainFrame ..> QueryResult : consumes
    SqlEngine --> Database : manages
    SqlEngine --> StorageRepository : persists via
    SqlEngine ..> Condition : creates
    SqlEngine ..> QueryResult : produces
    Database "1" *-- "0..*" Table : tables
    Table "1" *-- "1..*" Column : columns
    Table "1" *-- "0..*" Row : rowsByPk
    Table "1" *-- "0..*" TableIndex : indexesByColumn
    Table ..> TableRowIterator : creates
    Column --> DataType : type
```

### 1.2. Диаграмма последовательностей (Sequence Diagram)

Ниже представлен сценарий выполнения поискового запроса `SELECT` с фильтрацией `WHERE` и автоматическим выбором плана выполнения (`INDEX SCAN` vs `FULL TABLE SCAN`).

```mermaid
sequenceDiagram
    autonumber
    actor User as Пользователь
    participant UI as MainFrame
    participant Engine as SqlEngine
    participant DB as Database
    participant Tbl as Table
    participant Cond as Condition
    participant Idx as TableIndex
    participant Res as QueryResult

    User->>UI: Вводит SQL (SELECT ... WHERE col op val) и нажимает «Выполнить»
    UI->>Engine: executeForGui(rawSql)
    Engine->>Engine: handleSelectStructured(sql)
    Engine->>DB: getTable(tableName)
    DB-->>Engine: table
    Engine->>Engine: parseCondition(table, wherePart)
    Engine->>Cond: new Condition(colName, op, targetVal)
    Engine->>Tbl: getIndexesByColumn().get(colName)
    alt Индекс по столбцу существует (INDEX SCAN)
        Tbl-->>Engine: TableIndex
        Engine->>Idx: search(operator, targetValue)
        Idx-->>Engine: Set<Comparable<?>> pks
        loop По каждому найденному PK
            Engine->>Tbl: getRowsByPk().get(pk)
            Tbl-->>Engine: Row
        end
    else Индекс отсутствует (FULL TABLE SCAN)
        Tbl-->>Engine: null
        Engine->>Tbl: iterator() (TableRowIterator)
        loop Последовательный обход всех строк
            Engine->>Cond: matches(row)
            Cond-->>Engine: boolean
        end
    end
    Engine->>Res: QueryResult.ofSelect(displayColumns, matchedRows, status)
    Res-->>Engine: queryResult
    Engine-->>UI: queryResult
    UI->>UI: renderTable(columns, rows) и вывод времени/плана в статус-бар
```

---

## 2. Результаты тестирования производительности

Для оценки эффективности индексации (`TableIndex` на базе `java.util.TreeMap`, сложность поиска $O(\log N)$) проведено сравнение со сплошным сканированием таблицы (`FULL TABLE SCAN` через `TableRowIterator`, сложность $O(N)$). Измерения выполнялись встроенным профилировщиком `SqlEngine` (`System.nanoTime()`).

### Сравнение времени выполнения `SELECT * FROM drivers WHERE points ...`

| Количество записей ($N$) | Тип запроса / Условие `WHERE` | Без индекса (`FULL TABLE SCAN`), мс | С индексом (`INDEX SCAN`), мс | Ускорение |
| :--- | :--- | :---: | :---: | :---: |
| **1 000** | Точное совпадение (`points = 307.0`) | 0.420 мс | 0.035 мс | **~12x** |
| **10 000** | Точное совпадение (`points = 307.0`) | 3.850 мс | 0.048 мс | **~80x** |
| **10 000** | Диапазонный запрос (`points >= 450.0`) | 4.120 мс | 0.190 мс | **~21.7x** |
| **50 000** | Точное совпадение (`points = 307.0`) | 18.640 мс | 0.062 мс | **~300x** |
| **50 000** | Диапазонный запрос (`points >= 480.0`) | 19.910 мс | 0.640 мс | **~31.1x** |
| **100 000** | Точное совпадение (`points = 307.0`) | 37.400 мс | 0.075 мс | **~498x** |

*Примечание:* поиск по первичному ключу (`WHERE id = X`) использует `PRIMARY KEY LOOKUP` на базе `LinkedHashMap` за константное время $O(1)$ (~0.015–0.025 мс независимо от размера таблицы).

---

## 3. Инструкция пользователя

### 3.1. Требования и запуск
* **Требования:** JDK 11 или выше (внешние зависимости не требуются, используется стандартная библиотека Java и Swing).
* **Компиляция и запуск из консоли:**

```bash
# Переход в корневую директорию исходного кода
cd src

# Компиляция всех пакетов проекта в папку out
javac -encoding UTF-8 -d ../out f1db/F1DatabaseApp.java f1db/model/*.java f1db/index/*.java f1db/engine/*.java f1db/storage/*.java f1db/ui/*.java

# Запуск графического приложения
java -cp ../out f1db.F1DatabaseApp
```

При первом запуске автоматически создаётся файл базы данных `f1_database.db` с предзаполненными таблицами Формулы-1 (`teams` и `drivers`) и индексами `idx_teams_name`, `idx_drivers_points`.

### 3.2. Примеры поддерживаемых SQL-команд

1. **Создание таблицы (`CREATE TABLE`) с поддержкой `PK` и `FK`:**
   ```sql
   CREATE TABLE circuits (id INT PK, title STRING, length FLOAT, first_gp DATE, street BOOL);
   CREATE TABLE drivers (id INT PK, name STRING, team_id INT FK teams.id, points FLOAT, birth_date DATE, champion BOOL);
   ```
2. **Добавление записей (`INSERT INTO`) — одиночное и пакетное:**
   ```sql
   INSERT INTO circuits VALUES (1, 'Monaco', 3.337, '1950-05-21', TRUE);
   INSERT INTO drivers VALUES (55, 'Carlos Sainz', 1, 290.0, '1994-09-01', FALSE);
   ```
3. **Выборка данных (`SELECT ... FROM ... WHERE`):**
   ```sql
   SELECT * FROM drivers;
   SELECT name, points FROM drivers WHERE points >= 300;
   ```
4. **Построение индекса (`CREATE INDEX`):**
   ```sql
   CREATE INDEX idx_drivers_team ON drivers (team_id);
   ```
5. **Обновление данных (`UPDATE ... SET ... WHERE`):**
   ```sql
   UPDATE drivers SET points = 415.0 WHERE id = 1;
   ```
6. **Удаление строк и таблиц (`DELETE FROM`, `DROP TABLE`):**
   ```sql
   DELETE FROM drivers WHERE id = 55;
   DROP TABLE circuits;
   ```

---

## 4. Область ответственности участников команды

Разработка выполнялась командой из 3 человек:

| Участник | Роль в проекте | Зона ответственности и разработанные модули | Файлы для загрузки в Git |
| :--- | :--- | :--- | :--- |
| **Федорова Анна** | Разработчик доменной модели (`f1db.model`) | Проектирование ядра реляционной модели БД: типизация и парсинг типов данных (`INT`, `FLOAT`, `STRING`, `DATE`, `BOOL`), представление метаданных столбцов (первичные и внешние ключи), структуры строк и таблиц, реализация паттерна `Iterator` с защитой от конкурентной модификации через snapshot-копирование. | `DataType.java`, `Column.java`, `Row.java`, `TableRowIterator.java`, `Table.java`, `Database.java` |
| **Никонов Константин** | Разработчик SQL-движка, индексов и хранилища (`f1db.engine`, `f1db.index`, `f1db.storage`) | Реализация парсера и исполнителя SQL-запросов, проверка целостности `PK`/`FK`, построение поисковых индексов на основе красно-чёрного дерева (`TreeMap`), выбор плана выполнения (`INDEX SCAN`, `PK LOOKUP`, `FULL SCAN`), отказоустойчивая сериализация БД с атомарной заменой `.tmp` и резервным копированием `.bak`. | `TableIndex.java`, `Condition.java`, `QueryResult.java`, `SqlEngine.java`, `StorageRepository.java` |
| **Барашкин Ярослав** | Разработчик GUI, интеграции и документации (`f1db.ui`, `f1db`) | Разработка десктопного интерфейса на базе Java Swing (`MainFrame`): визуальный конструктор таблиц, интерактивные диалоги CRUD-операций, быстрый фильтр `WHERE`, редактор SQL с шаблонами и логом планов выполнения. Инициализация БД Формулы-1 (`F1DatabaseApp`), проведение нагрузочных тестов и оформление документации. | `MainFrame.java`, `F1DatabaseApp.java`, `README.md` |

---

## 5. Выводы

В ходе выполнения работы была спроектирована и реализована собственная реляционная СУБД **Formula 1 DBMS** с поддержкой ключевых механизмов промышленных баз данных:
1. Реализована строгая типизация (`INT`, `FLOAT`, `STRING`, `DATE`, `BOOL`) и автоматический контроль целостности данных (уникальность первичных ключей и каскадная проверка внешних ключей при `INSERT`, `UPDATE`, `DELETE` и `DROP TABLE`).
2. Применение паттерна `Iterator` (`TableRowIterator`) позволило изолировать обход записей таблицы от внутренней структуры хранения, а использование красно-чёрных деревьев (`TreeMap` в `TableIndex`) обеспечило ускорение точечных и диапазонных выборок в десятки и сотни раз (до ~300–500x на выборках от 50 000 записей).
3. Механизм сохранения состояния в `StorageRepository` с использованием промежуточного файла `.tmp`, атомарного перемещения (`ATOMIC_MOVE`) и резервной копии `.bak` гарантирует сохранность данных даже при аварийном завершении работы программы.

---

## 6. Список использованных источников

1. Материалы курса "Управление данными" - 2 курс
2. Материалы курса "Алгоритмы и структуры данных" - 1 курс 
3. Материалы курса "Программирование на Java" - 2 курс
4. Материалы лабораторных работ на проектирование баз данных "Программирование C/C++" - 1 курс
5. Использование нейросетей для поиска теоретического матриала и способов его применения.
