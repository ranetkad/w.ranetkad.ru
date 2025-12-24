# PostgreSQL - администрирование

## Коротко

Базовые операции: создание БД/пользователей, права, бэкапы.
Команды выполняются в psql или через утилиты pg_dump/pg_restore.

## Команды

Создание базы данных:

```sql
CREATE DATABASE mydb;
```

Удаление базы данных:

```sql
DROP DATABASE mydb;
```

Создание пользователя:

```sql
CREATE USER myuser WITH PASSWORD 'secret';
```

Выдать все права на БД:

```sql
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
```

Изменить пароль:

```sql
ALTER USER myuser WITH PASSWORD 'newpass';
```

Удалить пользователя:

```sql
DROP USER myuser;
```

## Примеры

Бэкап базы в SQL-файл:

```bash
pg_dump -d mydb -f backup.sql
```

Бэкап в сжатый формат (рекомендуется для больших БД):

```bash
pg_dump -d mydb -Fc -f backup.dump
```

Бэкап одной таблицы:

```bash
pg_dump -d mydb -t users -f users.sql
```

Восстановление из SQL-файла:

```bash
psql -d mydb -f backup.sql
```

Восстановление из сжатого формата:

```bash
pg_restore -d mydb backup.dump
```

Восстановление с пересозданием БД:

```bash
pg_restore -C -d postgres backup.dump
```

Флаг `-C` создаст БД перед восстановлением.

Параллельное восстановление (быстрее):

```bash
pg_restore -d mydb -j 4 backup.dump
```
