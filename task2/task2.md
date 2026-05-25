# Задание 2. Конфигурация Master-Slave репликации

## Стек

- MySQL 8.0 (два контейнера Docker)
- Master: порт 3316
- Slave: порт 3317

## Структура файлов

```
task2/
├── docker-compose.yml
├── master/
│   └── my.cnf
└── slave/
    └── my.cnf
```

## Конфигурация мастера (`master/my.cnf`)

```ini
[mysqld]
server-id     = 1        # уникальный ID узла
log_bin       = mysql-bin  # включаем бинарный лог
binlog_format = ROW      # формат — построчный
binlog_do_db  = repl_db  # реплицируем только эту БД
```

## Конфигурация слейва (`slave/my.cnf`)

```ini
[mysqld]
server-id  = 2           # отличается от мастера
relay_log  = relay-bin   # лог для хранения событий от мастера
read_only  = 1           # слейв только для чтения
```

## Шаги настройки

### 1. Запуск контейнеров

```bash
docker compose up -d
```

### 2. Создание пользователя репликации на мастере

```sql
CREATE USER 'repl_user'@'%' IDENTIFIED WITH mysql_native_password BY 'replpass';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS\G
```

### 3. Подключение слейва к мастеру

```sql
CHANGE MASTER TO
  MASTER_HOST='<IP мастера>',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='replpass',
  MASTER_LOG_FILE='mysql-bin.000003',
  MASTER_LOG_POS=851;
START SLAVE;
```

### 4. Проверка статуса слейва

```sql
SHOW SLAVE STATUS\G
```

Ключевые поля:
- `Slave_IO_Running: Yes` — слейв читает бинлог мастера
- `Slave_SQL_Running: Yes` — слейв применяет события
- `Seconds_Behind_Master: 0` — нет отставания

## Скриншоты

### Статус мастера (`SHOW MASTER STATUS`)

<img width="655" height="129" alt="Screenshot 2026-05-25 at 22 03 01" src="https://github.com/user-attachments/assets/b77aa425-7107-4a87-866b-5c5d5e714b27" />


### Статус слейва (`SHOW SLAVE STATUS`)

<img width="655" height="887" alt="Screenshot 2026-05-25 at 22 04 09" src="https://github.com/user-attachments/assets/4b463bdb-350c-4eaf-80eb-c6e06e99080d" />

### Данные на мастере

<img width="780" height="111" alt="Screenshot 2026-05-25 at 22 04 52" src="https://github.com/user-attachments/assets/02de72e2-4708-42fa-8841-42990fe0c3b6" />

### Данные на слейве (после репликации)

<img width="765" height="89" alt="Screenshot 2026-05-25 at 22 05 07" src="https://github.com/user-attachments/assets/d777b74f-d400-43a8-b763-99595b88e437" />

## Результат

Репликация настроена и работает:
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Seconds_Behind_Master: 0`
- Данные, записанные на мастере, появляются на слейве без задержки.
