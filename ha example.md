## Вступление

<p align="justify">Имеем сервер с установленным home assistant (HA), собирающим данные с умных устройств.
Нативно поддерживается только sqlite, что со временем будет тормозить систему.
Поэтому будем использовать postgresql. 
Для построения графиков добавим timescaledb. Данные туда будут поступать автоматически через стороннее расширение для HA - ltss, в соответствующую таблицу.</p>
Настроили, поехали.

## Проверяем данные

Первым делом смотрим структуру и типы данных. 

```shell
homeassistant_db=# 
SELECT column_name,
		data_type
FROM information_schema.columns
WHERE table_name = 'ltss';

 column_name |        data_type
-------------+--------------------------
 id          | bigint
 time        | timestamp with time zone
 entity_id   | character varying
 state       | character varying
 attributes  | jsonb
```

Для графиков нам нужно всего три переменных, посмотрим на них.
В качестве образца возьмем датчик фиксирующий общую потребляему мощность.

```sql
SELECT time,entity_id,state
FROM ltss
WHERE entity_id LIKE 'sensor.shellyem3_3494546ed3b3_channel_a_power'
GROUP BY "time",2,3
LIMIT 5;
```


```shell
             time              |                   entity_id                   | state
-------------------------------+-----------------------------------------------+--------
 2023-04-11 17:45:05.958863+03 | sensor.shellyem3_3494546ed3b3_channel_a_power | 189.82
 2023-04-11 17:45:08.095359+03 | sensor.shellyem3_3494546ed3b3_channel_a_power | 190.17
 2023-04-11 17:45:22.08828+03  | sensor.shellyem3_3494546ed3b3_channel_a_power | unavailable
 2023-04-11 17:45:23.08927+03  | sensor.shellyem3_3494546ed3b3_channel_a_power | 188.96
 2023-04-11 17:45:38.091343+03 | sensor.shellyem3_3494546ed3b3_channel_a_power | 189.4

```

У нас есть несколько проблемы:
- Очень частый опрос. Регистрировать пики надо, но мы хотим красивый исторический дашборд.
- Слишком длинные названия для сущностей.
- Переменная со значением, по которому будет строиться график - текстовая и может содержать состояние unavailable.

Сократим время до секунд [date_trunc](https://www.postgresql.org/docs/9.1/functions-datetime.html#FUNCTIONS-DATETIME-TRUNC).<br>
Уберем лишний текст из названий [regexp_replace](https://www.postgresqltutorial.com/postgresql-string-functions/regexp_replace/). <br>
Текст сущности [переводим](https://learnsql.com/cookbook/how-to-convert-a-string-to-a-numeric-value-in-postgresql/) в числовой формат, а чтобы не возникало ошибки ставим фильтр <>. Можно и здесь использовать регулярное выражение, выбирая числа, но в данном случае возможен лишь цифровой формат или состояние unavailable. 


```sql
SELECT date_trunc('second', "time") AS "time", 
		Regexp_replace(Regexp_replace(entity_id, 'sensor.', '', 'g'),'_3494546ed3b3','', 'g') AS "name", 
		state::DECIMAL AS "value"
FROM ltss
WHERE entity_id LIKE '%shellyem3_3494546ed3b3_channel_a_power'
		AND state <> 'unavailable'
		AND "time" >='2023-04-19'
GROUP BY "time",2,3
ORDER BY 1,2
```

## Строим графики

Можно строить график. Только вот отклонения будут краткосрочными, например включение чайника. Улучшим читаемость. Вспоминаем скользящее среднее.<br>
Для наглядности добавим смещение.


```sql
SELECT date_trunc('second', "time") AS "time", 
		Regexp_replace(Regexp_replace(entity_id, 'sensor.', '', 'g'),'_3494546ed3b3','', 'g') AS "name", 
		state::DECIMAL AS "value",
		round(AVG(state::DECIMAL) over w) +2000 as state_avg
FROM ltss
WHERE entity_id LIKE '%shellyem3_3494546ed3b3_channel_a_power'
		AND state <> 'unavailable'
		AND "time" >='2023-04-19'
GROUP BY "time",2,3
window w as (
		order by "time",2,3
		rows between 20 preceding and 20 following
)
ORDER BY 1,2;
```

![[graph_visualiser-1681898078971.png]](attachment/graph_visualiser-1681898078971.png)

Выглядит отвратительно.<br>
Давайте сделаем выборку по часам.


```sql
SELECT hour, AVG(state_avg) from(
SELECT date_trunc('second', "time") AS "time", 
		Regexp_replace(Regexp_replace(entity_id, 'sensor.', '', 'g'),'_3494546ed3b3','', 'g') AS "name", 
		state::DECIMAL AS "value",
		round(AVG(state::DECIMAL) over w) as state_avg,
		case extract(dow from time::timestamp)
		when 0 then 'Sunday'
		when 1 then 'Monday'
		when 2 then 'Tuesday'
		when 3 then 'Wednesday'
		when 4 then 'Thursday'
		when 5 then 'Friday'
		when 6 then 'Saturday'
		end dow,
		extract(hour from time::timestamp) as hour
FROM ltss
WHERE entity_id LIKE '%shellyem3_3494546ed3b3_channel_a_power'
		AND state <> 'unavailable'
		AND "time" >='2023-04-12'
GROUP BY "time",2,3
window w as (
		order by "time",2,3
		rows between 60 preceding and 60 following
)
ORDER BY 1,2)A
GROUP BY "hour"
order by "hour"
```

![[graph_visualiser-1681897799886.png]](attachment/graph_visualiser-1681897799886.png)

И по дням недели.<br>
Цифры в названиях для корректной сортировки без использования дополнительных вычислений.

```sql
SELECT dow, AVG(state_avg) from(
SELECT date_trunc('second', "time") AS "time", 
		Regexp_replace(Regexp_replace(entity_id, 'sensor.', '', 'g'),'_3494546ed3b3','', 'g') AS "name", 
		state::DECIMAL AS "value",
		round(AVG(state::DECIMAL) over w) as state_avg,
		case extract(dow from time::timestamp)
		when 0 then '7 Sunday'
		when 1 then '1 Monday'
		when 2 then '2 Tuesday'
		when 3 then '3 Wednesday'
		when 4 then '4 Thursday'
		when 5 then '5 Friday'
		when 6 then '6 Saturday'
		end dow,
		extract(hour from time::timestamp) as hour
FROM ltss
WHERE entity_id LIKE '%shellyem3_3494546ed3b3_channel_a_power'
		AND state <> 'unavailable'
		AND "time" >='2023-04-12'
GROUP BY "time",2,3
window w as (
		order by "time",2,3
		rows between 60 preceding and 60 following
)
ORDER BY 1,2)A
GROUP BY "dow"
order by "dow"
```

![[graph_visualiser-1681898894962.png]](attachment/graph_visualiser-1681898894962.png)

Графики и запросы делал в pgadmin4 для последующего экспорта в grafana. <br>
BI системы позволят не производить некоторые операции (grafana нет).

