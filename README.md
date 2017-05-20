# iqoption
Собеседование в IQ Option

## Вопросы
### Проектирование.
*5. Какой тип данных лучше использовать для хранения временных отметок и почему?*

Лучше всего сразу использовать *timestamp with time zone*, так как разницу по накладным расходам вы не заметите, а когда у вас сервера будут в разных географических точках планеты и с разными часовыми поясами, то не будет проблем с определением хонологии операций. У каждого сервера будет высталвен свой часовой пояс и timestamptz при сохранении будет преобразован в UTC.

### Программирование («Как будет выглядеть запрос?»)
*9. Из таблицы Вам нужно получить самые свежие новости в каждой из категорий (по одной новости
на категорию).*
```sql
create table news (newsid serial not null, catid integer not null, stamp timestamp not null);

insert into news(catid, stamp)
select 
  catid, 
  now() - random() * interval '7 days' as stamp 
from generate_series(1,3) as catid, generate_series(1,5) as num
order by catid, num;

select s.catid, max(s.newsid) from (
  select f.newsid, f.catid from (
    select 
      newsid, 
      catid, 
      stamp = max(stamp) over(partition by catid order by stamp) as checked 
    from news
  ) f where f.checked
) s group by s.catid;
```

*10. Вам нужно создать новую таблицу не из результата запроса SELECT, а из результата запроса
DELETE.*
```sql
select generate_series(1,10) as id into data;

create table deldata as
with deleted as (
  delete from data where id in (9,10) returning *
)
select * from deleted ;
```

*11. Сгруппировав таблицу по определённому критерию, Вам нужно для каждой группы получить
конкатенацию текстового поля в порядке добавления записей.*
```sql
create table datatxt (id serial not null, txt text not null, groupid integer not null, stamp timestamp not null);

insert into datatxt(txt, groupid, stamp)
select 
  'txt'||(random() * 100) as text, 
  groupid, 
  now() - random() * interval '10 days' as stamp 
from generate_series(1,3) as groupid, generate_series(1,5) as num
order by groupid, num;

with framed_data as (
  select
    row_number() over(partition by groupid order by stamp asc),
    groupid,
    string_agg(txt::text,',') over(partition by groupid order by stamp asc) as concut
  from datatxt
),
max_from_frames as (
  select groupid, max(row_number) as row_number from framed_data
  group by groupid
)
select m.groupid, f.concut from max_from_frames m
join framed_data f using(row_number, groupid);
```

*12. Отсортировав таблицу по определённому критерию, Вам нужно для каждой записи получить
среднее значение поля текущей и пяти предыдущих записей («скользящую среднюю»).*

Так как в условии не указано поведение для первых строк, у которых не набирается 5 предыдущих, использую штатное поведение оконной функции
``` sql
create table dataavg(id serial not null, value integer not null);

insert into dataavg(value)
select s.value from (
  select (random() * 100)::integer as value, generate_series(1,10)
) as s;

select id, value, avg(value) over(order by id asc rows 5 preceding)
from dataavg order by id asc;
```

### Внутреннее устройство.
*17. Как в PostgreSQL реализованы enum-типы? Можно ли удалить значение из enum-типа?*

```sql
create type test as enum('a','b');

select enumtypid, enumsortorder, enumlabel from pg_enum 
where enumtypid = (
  select oid from pg_type 
  where typname = 'test' and typnamespace = (
    select oid from pg_namespace where nspname = 'public'
  )
);

delete from pg_enum 
where enumtypid = (
  select oid from pg_type 
  where typname = 'test' and typnamespace = (
    select oid from pg_namespace where nspname = 'public'
  )
)
and enumlabel = 'a';
```

### Решение проблем.
*19. Программист Вася жалуется, что его запрос выполняется уже полчаса и не собирается
завершаться. Перечислите возможные причины этого и способы их устранения*

Подозреваю, что программист Василий запустил запрос, который обращается к ресурсам, которые держит блокировкой другой запрос, который или очень долгий, или просто кто-то не написал COMMIT. Надо залезть в pg_locks и разобраться, кого убиваем, кого оставляем, кого коммитим.

*20. Индекс, созданный полгода назад, успел вырасти в четыре раза при росте таблицы на 10%. Как
его вернуть к нормальным размерам и нужно ли?*

Очевидно, что шла активная работа с таблицей и из нее много удалялось и добавлялось записей. За счет механизмов внутренней очистки и удачно подобранного фактора заполнения страницы новые странички почти не создавались. А вот в индексе полно ссылок на dead ctid в заголовке станиц таблицы. Кстати, тут бы я проверил, работает ли hot обновление. Вообще с разросшимся индексом делать что-то надо. Как я понимаю, количество слоев индекса растет логарифмически и при поиске по равенству скорость не просядет. А вот при поиске по диапазону из индекса будут вытаскиваться указатели на мертвые строки, которые придется отбрасывать. Поэтому я бы создал копию этого индекса (если позволяет свободное место) в режиме CONCURRENTLY, проверил, что он ок по размеру и потом удалил бы старый (и для красоты переименовал новый индекс в старое имя).
