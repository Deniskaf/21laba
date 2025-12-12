# 21laba  
## Тема: 1 task

---

##   таблицы для БД 

 Таблица 1: `Peers` — Информация о пирах
```sql
CREATE TABLE IF NOT EXISTS Peers (
    Nickname VARCHAR PRIMARY KEY,
    Birthday DATE NOT NULL
);
```

---

 Таблица 2: `Tasks` — Задания
```sql
CREATE TABLE IF NOT EXISTS Tasks (
    Title VARCHAR PRIMARY KEY,
    ParentTask VARCHAR REFERENCES Tasks(Title),
    MaxXP INTEGER NOT NULL
);
```

---

 Таблица 3: `checks` — Проверки заданий
```sql
CREATE TABLE IF NOT EXISTS Checks (
    ID BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    Peer VARCHAR NOT NULL REFERENCES Peers(Nickname),
    Task VARCHAR NOT NULL REFERENCES Tasks(Title),
    Date DATE NOT NULL
);
```

---

 Таблица 4: `p2p` — P2P проверки
```sql
CREATE TABLE IF NOT EXISTS P2P (
    ID BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    CheckID BIGINT NOT NULL REFERENCES Checks(ID),
    CheckingPeer VARCHAR NOT NULL REFERENCES Peers(Nickname),
    State check_status NOT NULL,
    Time TIME NOT NULL
);
```

---

 Таблица 5: `verter` — Проверки Verter'ом
```sql
CREATE TABLE IF NOT EXISTS Verter (
    ID BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    CheckID BIGINT NOT NULL REFERENCES Checks(ID),
    State check_status NOT NULL,
    Time TIME NOT NULL
);
```

---

 Таблица 6: `transferredpoints` — Переданные пир-поинты
```sql
CREATE TABLE IF NOT EXISTS TransferredPoints (
    ID BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    CheckingPeer VARCHAR NOT NULL REFERENCES Peers(Nickname),
    CheckedPeer VARCHAR NOT NULL REFERENCES Peers(Nickname),
    PointsAmount INTEGER NOT NULL
);
```

---

 Таблица 7: `friends ` — Дружеские связи
```sql
CREATE TABLE IF NOT EXISTS Friends (
    ID BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    Peer1 VARCHAR NOT NULL REFERENCES Peers(Nickname),
    Peer2 VARCHAR NOT NULL REFERENCES Peers(Nickname),
    CHECK (Peer1 <> Peer2)
);
```

 Таблица 8: `recommendations ` — Рекомендации
```sql
CREATE TABLE IF NOT EXISTS Recommendations (
    ID BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    Peer VARCHAR NOT NULL REFERENCES Peers(Nickname),
    RecommendedPeer VARCHAR NOT NULL REFERENCES Peers(Nickname),
    CHECK (Peer <> RecommendedPeer)
);
```

---

 Таблица 9: `xp` — Опыт
```sql
CREATE TABLE IF NOT EXISTS XP (
    ID BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    CheckID BIGINT NOT NULL REFERENCES Checks(ID),
    XPAmount INTEGER NOT NULL
);
```

---

 Таблица 10: `timetracking ` —Отслеживание времени в кампусе
```sql
CREATE TABLE IF NOT EXISTS TimeTracking (
    ID BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    Peer VARCHAR NOT NULL REFERENCES Peers(Nickname),
    Date DATE NOT NULL,
    Time TIME NOT NULL,
    State INTEGER NOT NULL CHECK (State IN (1, 2)) -- 1 = вход, 2 = выход
);
```

---

![alt text](1.png)


---
## Тема: 1 и 2 task

## 3. 1. Процедура добавления P2P проверки

```sql
CREATE OR REPLACE PROCEDURE AddP2PCheck(
    IN CheckingPeer VARCHAR,      -- Никнейм проверяющего пира
    IN CheckedPeer VARCHAR,       -- Никнейм проверяемого пира
    IN TaskName VARCHAR,          -- Название задания
    IN CheckState check_status,   -- Статус проверки (Start, Success, Failure)
    IN CheckTime TIME            -- Время проверки
) AS $$
DECLARE
    CheckID BIGINT;               -- Переменная для хранения ID проверки
BEGIN
    -- Создаём новую запись в таблице Checks, если её ещё нет
    -- Используем CURRENT_DATE для автоматической установки текущей даты
    INSERT INTO Checks (Peer, Task, Date)
    VALUES (CheckedPeer, TaskName, CURRENT_DATE)
    ON CONFLICT DO NOTHING        -- Если запись уже существует, не вызываем ошибку
    RETURNING ID INTO CheckID;    -- Сохраняем ID вновь созданной записи

    -- Если запись уже существовала (CheckID IS NULL), находим её ID
    IF CheckID IS NULL THEN
        SELECT ID INTO CheckID
        FROM Checks
        WHERE Peer = CheckedPeer 
          AND Task = TaskName 
          AND Date = CURRENT_DATE; -- Ищем сегодняшнюю проверку
    END IF;

    -- Добавляем запись о P2P проверке в таблицу P2P
    INSERT INTO P2P (CheckID, CheckingPeer, State, Time)
    VALUES (CheckID, CheckingPeer, CheckState, CheckTime);
    
    -- Триггер автоматически обновит TransferredPoints при State = 'Start'
END;
$$ LANGUAGE plpgsql;
```

---
## 2. Процедура добавления проверки Verter'ом
```sql
CREATE OR REPLACE PROCEDURE AddVerterCheck(
    IN CheckedPeer VARCHAR,       -- Никнейм проверяемого пира
    IN TaskName VARCHAR,          -- Название задания
    IN CheckState check_status,   -- Статус проверки Verter'ом (Start, Success, Failure)
    IN CheckTime TIME            -- Время проверки
) AS $$
DECLARE
    CheckID BIGINT;               -- Переменная для хранения ID проверки
BEGIN
    -- Находим ID последней (самой свежей) проверки для этого пира и задания
    -- Сортируем по дате и ID в убывающем порядке, берём первую запись
    SELECT c.ID INTO CheckID
    FROM Checks c
    WHERE c.Peer = CheckedPeer 
      AND c.Task = TaskName
    ORDER BY c.Date DESC, c.ID DESC
    LIMIT 1;

    -- Добавляем запись о проверке Verter'ом
    -- предполагается, что P2P проверка уже завершена успешно
    INSERT INTO Verter (CheckID, State, Time)
    VALUES (CheckID, CheckState, CheckTime);
END;
$$ LANGUAGE plpgsql;
```

## 3. Триггер для автоматического обновления TransferredPoints
```sql
-- Функция, которая будет вызываться триггером
CREATE OR REPLACE FUNCTION UpdateTransferredPoints()
RETURNS TRIGGER AS $$
BEGIN
    -- Обновляем TransferredPoints только при начале новой P2P проверки
    IF NEW.State = 'Start' THEN
        -- Вставляем или обновляем запись о передаче пир-поинтов
        INSERT INTO TransferredPoints (CheckingPeer, CheckedPeer, PointsAmount)
        VALUES (
            NEW.CheckingPeer,                    -- Кто проверяет
            (SELECT Peer FROM Checks WHERE ID = NEW.CheckID), -- Кого проверяют
            1                                    -- Добавляем 1 поинт
        )
        ON CONFLICT (CheckingPeer, CheckedPeer)  -- Если пара уже существует
        DO UPDATE SET PointsAmount = TransferredPoints.PointsAmount + 1; -- Увеличиваем счётчик
    END IF;
    RETURN NEW;  -- Возвращаем новую запись для последующих триггеров
END;
$$ LANGUAGE plpgsql;

-- Создаём триггер, который срабатывает после каждой вставки в таблицу P2P
DROP TRIGGER IF EXISTS AfterP2PInsert ON P2P;
CREATE TRIGGER AfterP2PInsert
AFTER INSERT ON P2P                    -- Срабатывает после INSERT
FOR EACH ROW                          -- Для каждой вставленной строки
EXECUTE FUNCTION UpdateTransferredPoints(); -- Выполняет нашу функцию

```

код создания тригера 
```sql
DROP TRIGGER IF EXISTS AfterP2PInsert ON P2P;
CREATE TRIGGER AfterP2PInsert
AFTER INSERT ON P2P
FOR EACH ROW
EXECUTE FUNCTION UpdateTransferredPoints();
```
![alt text](2.png)
тестовые данные для таблиц 
```sql
-- Очистка таблиц 
TRUNCATE TABLE Peers, Tasks, Checks, P2P, Verter, TransferredPoints 
RESTART IDENTITY CASCADE;

-- 1. Добавляем пиров
INSERT INTO Peers (Nickname, Birthday) VALUES
('john_doe', '1995-05-15'),
('jane_smith', '1998-08-22'),
('alex_wong', '1997-03-10'),
('maria_garcia', '1999-11-30'),
('peter_pan', '2000-01-05');

-- 2. Добавляем задания (учитываем зависимости между ними)
INSERT INTO Tasks (Title, ParentTask, MaxXP) VALUES
('C2_SimpleBashUtils', NULL, 250),
('C3_s21_string+', 'C2_SimpleBashUtils', 500),
('C5_s21_decimal', 'C3_s21_string+', 350),
('CPP1_s21_matrix+', 'C5_s21_decimal', 300),
('A1_Maze', 'CPP1_s21_matrix+', 300);
```
Тестирование процедуры AddP2PCheck
```sql
-- Тест 1: Джон проверяет Джейн по заданию C2_SimpleBashUtils
CALL AddP2PCheck('john_doe', 'jane_smith', 'C2_SimpleBashUtils', 'Start', '10:00:00');
CALL AddP2PCheck('john_doe', 'jane_smith', 'C2_SimpleBashUtils', 'Success', '10:30:00');

-- Проверяем результат:
SELECT * FROM Checks;
SELECT * FROM P2P;
SELECT * FROM TransferredPoints; -- Должна появиться запись +1 поинт

-- Тест 2: Алекс проверяет Марию по тому же заданию
CALL AddP2PCheck('alex_wong', 'maria_garcia', 'C2_SimpleBashUtils', 'Start', '11:00:00');
CALL AddP2PCheck('alex_wong', 'maria_garcia', 'C2_SimpleBashUtils', 'Failure', '11:20:00');

-- Тест 3: Проверка того же пира в тот же день (должна использовать существующий CheckID)
CALL AddP2PCheck('peter_pan', 'maria_garcia', 'C2_SimpleBashUtils', 'Start', '14:00:00');

-- Проверяем количество записей:
SELECT COUNT(*) as total_checks FROM Checks; -- Должно быть 3 (не 4!)
SELECT COUNT(*) as total_p2p FROM P2P;       -- Должно быть 5
```
Тестирование процедуры AddVerterCheck
```sql
-- Тест 1: Verter проверяет успешную работу Джейн
CALL AddVerterCheck('jane_smith', 'C2_SimpleBashUtils', 'Start', '10:35:00');
CALL AddVerterCheck('jane_smith', 'C2_SimpleBashUtils', 'Success', '10:36:00');

-- Тест 2: Verter проверяет неудачную работу Марии
CALL AddVerterCheck('maria_garcia', 'C2_SimpleBashUtils', 'Start', '11:25:00');
CALL AddVerterCheck('maria_garcia', 'C2_SimpleBashUtils', 'Failure', '11:26:00');

-- Проверяем результат:
SELECT * FROM Verter;
```
Тестирование триггера на TransferredPoints
```sql
-- Проверяем автоматическое обновление TransferredPoints:
SELECT * FROM TransferredPoints ORDER BY PointsAmount DESC;

-- Дополнительный тест: повторная проверка того же пира
CALL AddP2PCheck('john_doe', 'jane_smith', 'C3_s21_string+', 'Start', '15:00:00');

-- Проверяем, что PointsAmount увеличился до 2:
SELECT * FROM TransferredPoints 
WHERE CheckingPeer = 'john_doe' AND CheckedPeer = 'jane_smith';

-- Тест с разными комбинациями проверок
CALL AddP2PCheck('jane_smith', 'john_doe', 'C2_SimpleBashUtils', 'Start', '16:00:00');
CALL AddP2PCheck('maria_garcia', 'alex_wong', 'C3_s21_string+', 'Start', '17:00:00');

-- Итоговая проверка всех передач поинтов
SELECT 
    CheckingPeer, 
    CheckedPeer, 
    PointsAmount,
    CASE 
        WHEN CheckingPeer = CheckedPeer THEN 'ERROR: Самопроверка'
        ELSE 'OK'
    END as Validation
FROM TransferredPoints;
```
Комплексный тестовый сценарий
```sql
-- Сценарий: Полный цикл проверки с P2P и Verter
-- 1. Джейн проверяет задание C3_s21_string+ у Джона
CALL AddP2PCheck('jane_smith', 'john_doe', 'C3_s21_string+', 'Start', '09:00:00');
CALL AddP2PCheck('jane_smith', 'john_doe', 'C3_s21_string+', 'Success', '09:45:00');

-- 2. Verter проверяет код Джона
CALL AddVerterCheck('john_doe', 'C3_s21_string+', 'Start', '09:50:00');
CALL AddVerterCheck('john_doe', 'C3_s21_string+', 'Success', '09:51:00');

-- 3. Проверяем все таблицы
SELECT 'Checks' as table_name, COUNT(*) as row_count FROM Checks
UNION ALL
SELECT 'P2P', COUNT(*) FROM P2P
UNION ALL
SELECT 'Verter', COUNT(*) FROM Verter
UNION ALL
SELECT 'TransferredPoints', COUNT(*) FROM TransferredPoints;

-- 4. Просмотр детальной информации
SELECT 
    p.Nickname as CheckedPeer,
    c.Task,
    p2p.CheckingPeer,
    p2p.State as P2P_State,
    p2p.Time as P2P_Time,
    v.State as Verter_State,
    v.Time as Verter_Time,
    tp.PointsAmount
FROM Checks c
JOIN Peers p ON c.Peer = p.Nickname
JOIN P2P p2p ON c.ID = p2p.CheckID
LEFT JOIN Verter v ON c.ID = v.CheckID
LEFT JOIN TransferredPoints tp ON p2p.CheckingPeer = tp.CheckingPeer 
    AND p.Nickname = tp.CheckedPeer
ORDER BY c.Date, c.ID;
```
