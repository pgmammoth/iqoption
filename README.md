# iqoption
Собеседование в IQ Option

## Вопросы
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
