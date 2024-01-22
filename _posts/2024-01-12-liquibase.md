---
layout: post
title: Liquibase
categories: Инструменты
---

Как произвести изменение схемы базы данных в работающем приложении?

Как добавить новую таблицу или новую колонку в таблицу?

Как обеспечить переносимость изменений, если приложение развернуто в нескольких средах: в контуре разработки, 
тестирования, и промышленном контуре?

Различные подходы описаны, например, в [подробной статье](https://habr.com/ru/articles/330662/) на хабре.

На одном из проектов, где я работала, чтобы иметь возможность добавлять новые атрибуты по мере разработки, 
была огромная таблица, куда для всех объектов добавлялись новые атрибуты и их значения, и был написан собственный ORM 
фреймворк.

На текущем проекте используем систему управления версионированием баз данных **Liquibase**.

Изменение БД описывается в changeset, которые собираются в changelog.
Поддерживаются форматы XML, Yaml, JSON, SQL.

Для примера создадим файл changelog.xml и customer_table.sql, где опишем создание таблицы клиентов приложения.

_changelog.xml_
```changelog.xml
<databasechangeLog>
    <include author="darya"
             file="db/customer_table.sql"/>
</>
```
_customer_table.sql_
```customer_table.sql

create table customer
(
    id numeric
    name varchar2(200)
    surname varchar2(200)    
)
```

При запуске приложения Liquibase выполняет changeset и записывает его хэш таблицу databasechangelog.

Теперь допустим, нам нужно добавить в таблицу customer атрибут email.
Если просто изменить changeset, то при перезапуске приложения его хэш изменится, и приложение упадет с 
ошибкой.

Необходимо написать новый changeset, модифицирующий таблицу.

_update_customer_table.sql_

```update_customer_table.sql
alter table customer
add email varchar2
```
_changelog.xml_
```changelog.xml
<databasechangeLog>
    <include author="darya"
             file="db/customer_table.sql"/>
    <include author="darya"
             file="db/update_customer_table.sql"/>
</>
```


Но если приложение еще не поставлено на тестовый\промышленный контур, скорее всего будет предпочтительнее не создавать 
новый changeset, чтобы сохранить читаемость наших миграций. 

Можно удалить changeset с 
модифицируемой таблицой из databasechangelog перед запуском приложения и просто дописать email в create в customer_table.sql.