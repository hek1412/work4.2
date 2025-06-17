# work4.2


CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT,
    email TEXT,
    role TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE users_audit (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by TEXT,
    field_changed TEXT,
    old_value TEXT,
    new_value TEXT
);
-- Создаем функцию логирования
CREATE OR REPLACE FUNCTION log_user_changes()
RETURNS TRIGGER AS $$
BEGIN
    -- Логируем изменения имени
    IF OLD.name IS DISTINCT FROM NEW.name THEN
        INSERT INTO users_audit(user_id, changed_by, field_changed, old_value, new_value)
        VALUES (OLD.id, current_user, 'name', OLD.name, NEW.name);
    END IF;
    
    -- Логируем изменения email
    IF OLD.email IS DISTINCT FROM NEW.email THEN
        INSERT INTO users_audit(user_id, changed_by, field_changed, old_value, new_value)
        VALUES (OLD.id, current_user, 'email', OLD.email, NEW.email);
    END IF;
    
    -- Логируем изменения роли
    IF OLD.role IS DISTINCT FROM NEW.role THEN
        INSERT INTO users_audit(user_id, changed_by, field_changed, old_value, new_value)
        VALUES (OLD.id, current_user, 'role', OLD.role, NEW.role);
    END IF;
    
    -- Обновляем метку времени изменения
--    NEW.updated_at = CURRENT_TIMESTAMP;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Создаем тригер
CREATE TRIGGER trigger_log_user_changes
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION log_user_changes();


INSERT INTO users (name, email, role)
VALUES 
('Ivan Ivanov', 'ivan@example.com', 'director'),
('Anna Petrova', 'anna@example.com', 'manager');

-- Обновляем данные
UPDATE users
SET role = 'HR'
WHERE name = 'Ivan Ivanov';

-- Смотрим таблицы
SELECT * FROM users;
SELECT * FROM users_audit;

-- Если нужно разрешить запись в /tmp из PostgreSQL
ALTER SYSTEM SET cron.database_name = 'example_db';
ALTER SYSTEM SET cron.log_run = 'on';


-- Создаем функцию export_users()
CREATE OR REPLACE FUNCTION export_users()
-- Функция не возвращает значений
RETURNS VOID AS $$
DECLARE
    -- Переменная для хранения пути к экспортируемому файлу
    export_filename TEXT;
    -- Переменная для хранения количества экспортированных строк
    rows_exported INTEGER;
    -- Переменная для формирования сообщений об ошибках
    error_message TEXT;
    -- Переменная для хранения текста SQL-команды COPY
    copy_command TEXT;
BEGIN
    -- Формируем имя файла в формате: /tmp/users_audit_export_ГГГГ-ММ-ДД.csv
    export_filename := '/tmp/users_audit_export_' || to_char(CURRENT_DATE, 'YYYY-MM-DD') || '.csv';
    
    -- Выводим информационное сообщение о начале экспорта
    RAISE NOTICE 'Начало экспорта аудит-данных в файл: %', export_filename;
    
    -- Начало блока с обработкой исключений
    BEGIN
        -- Формируем текст SQL-команды COPY:
        -- 1. Выбираем все записи из users_audit за текущий день
        -- 2. Сортируем по дате изменения
        -- 3. Экспортируем в CSV файл с заголовками столбцов
        copy_command := '
            COPY (
                SELECT * FROM users_audit 
                WHERE changed_at >= CURRENT_DATE
                AND changed_at < CURRENT_DATE + INTERVAL ''1 day''
                ORDER BY changed_at
            ) TO ''' || export_filename || ''' WITH CSV HEADER';
        
        -- Выполняем сформированную команду
        EXECUTE copy_command;
        
        -- Получаем количество обработанных строк
        GET DIAGNOSTICS rows_exported = ROW_COUNT;
        
        -- Выводим сообщение об успешном завершении экспорта
        RAISE NOTICE 'Успешно экспортировано % строк в файл: %', rows_exported, export_filename;
        
    -- Блок обработки исключений
    EXCEPTION
        -- Обработка ошибки недостатка прав
        WHEN insufficient_privilege THEN
            error_message := 'Ошибка: Нет прав на запись в файл ' || export_filename;
            RAISE EXCEPTION '%', error_message;
            
        -- Обработка всех остальных ошибок    
        WHEN others THEN
            -- Формируем сообщение об ошибке с деталями из PostgreSQL
            error_message := 'Ошибка при экспорте: ' || SQLERRM;
            -- Выводим сообщение об ошибке с указанием проблемного файла
            RAISE EXCEPTION '% Исходный файл: %', error_message, export_filename;
    END;
END;
-- Завершение функции, указание языка PL/pgSQL
$$ LANGUAGE plpgsql;

-- Вызываем функцию
SELECT export_users();

-- Проверка установленного расширения
SELECT extname FROM pg_extension WHERE extname = 'pg_cron';

--Установка расширения
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Настройка ежедневного задания
SELECT cron.schedule(
    'nightly_audit_export',
    '0 3 * * *', -- Каждый день в 3:00
    $$SELECT export_users()$$
);

-- Просмотр активных задач cron
SELECT * FROM cron.job;

-- Удаление задачи cron
SELECT cron.unschedule('nightly_audit_export');

-- Удаление cron
DROP EXTENSION pg_cron;
    $$SELECT export_users()$$
);
-- Просмотр активных задач cron
SELECT * FROM cron.job;
