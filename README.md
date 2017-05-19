# iqoption
Собеседование в IQ Option

## Вопросы
*9. Из таблицы Вам нужно получить самые свежие новости в каждой из категорий (по одной новости
на категорию).*
```sql
create table news (newsid serial, catid integer, stamp timestamp);

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
