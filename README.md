# work4.2

--CREATE TABLE users (
--    id SERIAL PRIMARY KEY,
--    name TEXT,
--    email TEXT);
--
--CREATE TABLE users_history (
--    id SERIAL,
--    user_id INT,
--    old_name TEXT,
--    old_email TEXT,
--    changed_at TIMESTAMP DEFAULT now()
--);

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

CREATE TRIGGER trigger_log_user_changes
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION log_user_changes();

--CREATE OR REPLACE FUNCTION log_user_update()
--RETURNS TRIGGER AS $$
--BEGIN
--    INSERT INTO users_history(user_id, old_name, old_email)
--    VALUES (OLD.id, OLD.name, OLD.email);
--    RETURN NEW;
--END;
--$$ LANGUAGE plpgsql;
--
--CREATE TRIGGER trigger_log_user_update
--BEFORE UPDATE ON users
--FOR EACH ROW
--EXECUTE FUNCTION log_user_update();

INSERT INTO users (name, email, role)
VALUES 
('Ivan Ivanov', 'ivan@example.com', 'director'),
('Anna Petrova', 'anna@example.com', 'manager');

--INSERT INTO users (name, email)
--VALUES 
--('Ivan Ivanov', 'ivan@example.com'),
--('Anna Petrova', 'anna@example.com');
--
UPDATE users
SET role = 'HR'
WHERE name = 'Ivan Ivanov';

SELECT * FROM users;


-- Если нужно разрешить запись в /tmp из PostgreSQL
ALTER SYSTEM SET cron.database_name = 'example_db';
ALTER SYSTEM SET cron.log_run = 'on';



CREATE OR REPLACE FUNCTION export_users()
RETURNS VOID AS $$
DECLARE
    export_filename TEXT;
    rows_exported INTEGER;
    error_message TEXT;
    copy_command TEXT;
BEGIN
    -- Формируем имя файла с текущей датой
    export_filename := '/tmp/users_audit_export_' || to_char(CURRENT_DATE, 'YYYY-MM-DD') || '.csv';
    
    RAISE NOTICE 'Начало экспорта аудит-данных в файл: %', export_filename;
    
    BEGIN
        -- Создаем полную команду COPY с подставленным именем файла
        copy_command := '
            COPY (
                SELECT * FROM users_audit 
                WHERE changed_at >= CURRENT_DATE
                AND changed_at < CURRENT_DATE + INTERVAL ''1 day''
                ORDER BY changed_at
            ) TO ''' || export_filename || ''' WITH CSV HEADER';
        
        -- Выполняем команду
        EXECUTE copy_command;
        
        -- Получаем количество экспортированных строк
        GET DIAGNOSTICS rows_exported = ROW_COUNT;
        
        RAISE NOTICE 'Успешно экспортировано % строк в файл: %', rows_exported, export_filename;
        
    EXCEPTION
        WHEN insufficient_privilege THEN
            error_message := 'Ошибка: Нет прав на запись в файл ' || export_filename;
            RAISE EXCEPTION '%', error_message;
            
        WHEN others THEN
            error_message := 'Ошибка при экспорте: ' || SQLERRM;
            RAISE EXCEPTION '% Исходный файл: %', error_message, export_filename;
    END;
END;
$$ LANGUAGE plpgsql;


SELECT export_users();

SELECT extname FROM pg_extension WHERE extname = 'pg_cron';
CREATE EXTENSION IF NOT EXISTS pg_cron;
-- Задача будет выполняться ежедневно в 3:00 ночи
SELECT cron.schedule(
    'nightly_audit_export',
    '0 3 * * *', -- Каждый день в 3:00
    $$SELECT export_users()$$
);
-- Просмотр активных задач cron
SELECT * FROM cron.job;
