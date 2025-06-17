# work4.2

После создания таблиц и тригера, встовляем данные, проверяем, что данные записаны в БД

```
SELECT * FROM users;
```
![image](https://github.com/user-attachments/assets/5987ca09-b042-446f-a1e4-5e7e4881cc30)

Обновляем данные
```
UPDATE users
SET role = 'HR'
WHERE name = 'Ivan Ivanov';
```
Смотрим таблицы
```
SELECT * FROM users;
```
![image](https://github.com/user-attachments/assets/3ffd8b9f-cbe8-47f4-959d-751de9b1f1c9)
```
SELECT * FROM users_audit;
```
![image](https://github.com/user-attachments/assets/87baa63e-d07d-4e56-9f5f-182fb77778b9)

Тригер успешно выполнился, данные в users_audit внесены.


Создаем функцию export_users()
И взываем её
```
SELECT export_users();
```
![image](https://github.com/user-attachments/assets/acc06667-d99d-4f34-9043-5867a53fd439)

Теперь заходим в контейнер БД, проверяем созданный файл
```
docker exec -it postgres_db bash
ls /tmp
cat /tmp/users_audit_export_2025-06-17.csv 
```
![image](https://github.com/user-attachments/assets/121fae2b-8f45-41f4-b471-ca2cb5e710cb)

Устанавливаем расширение pg_cron и создаем ежедневное задание
```
CREATE EXTENSION IF NOT EXISTS pg_cron;
```
```
SELECT cron.schedule(
    'nightly_audit_export',
    '0 3 * * *', -- Каждый день в 3:00
    $$SELECT export_users()$$
);
```
Проверяем активные задачи cron
```
SELECT * FROM cron.job;
```
![image](https://github.com/user-attachments/assets/430fc461-c1ed-402d-bf68-4a3508e9960f)


