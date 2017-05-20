# iqoption
Собеседование в IQ Option

## Вопросы
### Общие вопросы.
*2. Что такое autovacuum?*

Автовакуум физически представляет из себя постоянный фоновый процесс autovacuum launcher, который мониторит изменения в таблицах (при запуске вакуума по регламенту мы не знаем, изменилось у нас пара строк или всю таблицу на три раза переписали, а тут за этим следит отдельный процесс). По определенным условиям в настройках PG он запускает рабочие процессы автоочистки autovacuum worker (вернее, просит postmaster запустить их). Есть много ручек, которые позволяют управлять частотой запуска, количеством процессов, временем работы и нагрузкой на систему и они подбираются эмпиричеки. Есть возможность настраивать эти ручки для каждой таблицы отдельно. Каждый рабочий процесс автоочистки занимается VACUUM, ANALYZE и FREEZE, то есть удалением ненужных кортежей из таблиц и индексов по причине MVCC, сбором статистики и обновлением карты видимости.

*3. Чем отличаются команды SET и SET LOCAL?*

Команда SET позволяет выставлять "на лету" значения многих параметров PG (по факту из postgresql.conf). Правда, для части параметров нужен суперпользователь, некоторые нельзя изменять после перезапуска сервера... А разница между SET (он же SET SESSION) и SET LOCAL в том, что первый выставляет значения параметров для сессии, а второй для транзакции (тразакция закончилась - значение вернулось в исходное для сессии).

*4. Как в PostgreSQL реализуется пул соединений?*

Нативно в PG нет пула соединений и каждый запрос будет открывать свое соединение. Так как соединения в PG достаточно ресурсоемки, то между приложением и базой втыкают прослойку в виде балансировщика. Для него прописывается максимальное количество соединений, которые он может держать открытыми к базе, и все запросы из приложения будут завернуты в данный пул соединений. Я знаю, что все хвалят pgBouncer от ребят из Skype, но у нас в проекте исспользуется асинхронная обертка над psycopg2 под названием momoko для питоновского tornado (так как больше с базой другие приложения не работали, только модули нашего приложения).

### Проектирование.
*5. Какой тип данных лучше использовать для хранения временных отметок и почему?*

Лучше всего сразу использовать *timestamp with time zone*, так как разницу по накладным расходам вы не заметите, а когда у вас сервера будут в разных географических точках планеты и с разными часовыми поясами, то не будет проблем с определением хонологии операций. У каждого сервера будет высталвен свой часовой пояс и timestamptz при сохранении будет преобразован в UTC.

*6. Какие типы индексов поддерживает PostgreSQL, чем они отличаются?*

Нативно:
+ B-дерево - классическое сбалансированное дерево, подходит для большинства типовых операций. Не умеет специфичные вещи вроде wildcard-поиска ("~"), ближайших соседей. Использование индекса в запросе зависит от порядка колонок в определении индекса.
+ хеш - никогда с ним не работал. Старый как бивень мамонта, не поддерживает обобщенный WAL, умеет только проверку по равенству. Устарел, не рекомендуется к использованию.
+ GiST - фреймворк для построения R-деревьев с различными классами операторов. Используется для поиска ближайших объектов, крайне полезен  при работе с геоданными. Как и GIN используется для нечеткого поиска (медленнее поиск, быстрее вставка).
+ SP-GiST - никогда с ним не работал. Знаю только, что это вариация GiST с учетом разбиения пространства на сегменты. И если окно поиска лежит в сегменте, то работает очень быстро. Используется для несбалансированных R-деревьев.
+ GIN - обратный инвертированный индекс. По факту представляет из себя "ключ - массив/B-дерево значений". Используется в нечетком и полнотекстовом поиске.
+ BRIN - никогда с ним не работал. Знаю, что он используется для очень больших таблиц, где данные лежат в достаточно отсортированном виде, (например, гигантские таблицы логов, строки в которых вставлялись последовательно по времени). Размечает данные по блоковым зонам и осуществляет поиск только в подходящей блоковой зоне (похоже по принципу на партицирование). Требует перепроверки найденных результатов. Так как индекс получается очень компактный, поиск очень быстрый.

Расширения:
+ RUM - тот же GIN, но в листьях можем хранить что-то полезное. Когда обрастет достаточным количесвом классов операторов, будет перкрасен.
+ BLOOM - никогда с ним не работал. Знаю, что он позволяет строить индекс по нескольким колонкам и будет использоваться незавиимо от порядка колонок в выражении запроса. Немного медленнее B-дерева, может только проверки на равенство.

*7. Зачем в PostgreSQL существуют схемы, и как их можно эффективно использовать?*

В первую очередь, схемы существуют, чтобы не запутаться, что где лежит)) А во вторую - для выставления ограничений на использование содержащихся в схеме объектов для различных групп пользователей PostgreSQL. Не стоит использовать схему public для своих объектов - ее используют расширения, как стандартную, и создают там горы мусора. Вообще тяга к порядку привела меня к ряду правил для наименования схем и храниния в них только определенного типа содержимого - для локальных объектов, для внешних методов (хранимых процедур и представлений, которые дергаются снаружи программой), аудита, тестов, импорта данных. Но это я отвлекся...

*8. Какие индексы Вы бы добавили/поменяли в данной таблице, а какие не стали бы?*
```sql
TABLE article
 article_id SERIAL NOT NULL PRIMARY KEY,
 author_id INT NOT NULL FOREIGN KEY,
 is_draft BOOL NOT NULL DEFAULT TRUE,
 title TEXT NOT NULL UNIQUE,
 text TEXT NOT NULL
```
Вопрос немного некорректный, так как индексы просто так не добавляют для красоты, а подстраивают под конкретные запросы. Но могу придумать ряд вероятных сценариев и сказать, какие индексы возможно потребовались бы.
+ У нас есть первичный ключ, так что по article_id мы статью всегда найдем. Тут ок.
+ Может возникнуть кейс, что нужно найти все статьи автора и отфильтровать из них черновики/готовые статьи.
```sql
create index author_draft_idx on article (author_id, is_draft);
```
Возможно, таблица будет оооочень большая и мы захотим вообще исключить затрагивание хипа, тогда можно будет сделать индекс покрывающим и активно обновлять таблицу видимости через freeze
```sql
create index author_draft_idx on article (author_id, is_draft, article_id);
```
+ Может потребоваться нечеткий поиск по заголовку. UNIQUE создал простой индекс типа B-дерево, он будет работать как ограничение, но для wildcard или нечеткого поиска не подходит.
```sql
create extension pg_trgm;
create index title_idx on article using gin(title gin_trgm_ops);
```
+ Может потребоваться полнотекстовый поиск по содержимому статьи. Тогда нужно создать в таблице дополнительное поле типа tsvector на базе text (использовав нужные словари), навесить триггер для согласованности text и нового поля и создать простой gin индекс.


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

*18. В каких случаях PostgreSQL использует временные файлы? Как можно оптимизировать работу с
временными файлами?*

Временные файлы создаются при работе с временными таблицами, сортировках и работе с хеш-таблицами. Оптимизировать работу для разных объектов по-разному.
+ Временные таблицы

Перестать их использовать и перейти на CTE. Плюс многие функции, которые их использовали пожно будет пометить STABLE, а не VOLATILE. Для ограничения суммарного объема временных файлов в рамках одного сеанса, создаваемых временными таблицами, есть параметр temp_file_limit.
+ Сортировки

Используются в ORDER BY, DISTINCT и соединениях слиянием, значит вначале неплохо посмотреть эти участки кода в самых тяжелых запросах. А еще можно подкрутить work_mem.
+ Хеш-таблицы

Используются при соединениях и агрегировании по хешу, а также обработке подзапросов IN с применением хеша. Опять смотрим самые тяжелые запросы по части кода в аггрерациях, слияниях и подзапросах. Опять же, крутим work_mem.


### Решение проблем.
*19. Программист Вася жалуется, что его запрос выполняется уже полчаса и не собирается
завершаться. Перечислите возможные причины этого и способы их устранения*

Подозреваю, что программист Василий запустил запрос, который обращается к ресурсам, которые держит блокировкой другой запрос, который или очень долгий, или просто кто-то не написал COMMIT. Надо залезть в pg_locks и разобраться, кого убиваем, кого оставляем, кого коммитим.

*20. Индекс, созданный полгода назад, успел вырасти в четыре раза при росте таблицы на 10%. Как
его вернуть к нормальным размерам и нужно ли?*

За счет механизмов внутренней очистки (HOT) и удачно подобранного фактора заполнения страницы новые странички почти не создавались. А вот в B-дереве индекса HOT нет и странички делились. Да, они очищались вакуумом, но их стало много (у PG почти нет возможностей отдать страницы назад), вот и стоят полупустые. С разросшимся индексом надо что-то делать - для получения данных придется загружать много полупустых страниц в общие буферы, что неэффективно. Да и индекс занимает много места на диске. Поэтому я бы создал копию этого индекса (если позволяет свободное место) в режиме CONCURRENTLY, проверил, что он уменьшился по размеру и удалил старый (для красоты переименовал новый индекс в старое имя).
