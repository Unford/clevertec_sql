# clevertec SQL
[Задание](task.sql)

### Вывести к каждому самолету класс обслуживания и количество мест этого класса
 ```postgresql
SELECT a.aircraft_code, s.fare_conditions, count(*)
from aircrafts_data a
         LEFT JOIN seats s on a.aircraft_code = s.aircraft_code
GROUP BY a.aircraft_code, s.fare_conditions;
```


### Найти 3 самых вместительных самолета (модель + кол-во мест)
 ```postgresql
SELECT a.model, count(*) seats_count
from aircrafts_data a
         left join seats s on a.aircraft_code = s.aircraft_code
group by a.aircraft_code
order by seats_count desc
limit 3;
```


### Вывести код, модель самолета и места не эконом класса для самолета 'Аэробус A321-200' с сортировкой по местам
 ```postgresql
SELECT ad.aircraft_code, ad.model, s.*
from aircrafts_data ad
         left join seats s on ad.aircraft_code = s.aircraft_code
where ad.model::json ->> 'ru' = 'Аэробус A321-200'
  and s.fare_conditions != 'Economy'
order by s.seat_no;
```


### Вывести города в которых больше 1 аэропорта ( код аэропорта, аэропорт, город)
 ```postgresql
SELECT ad.airport_code, ad.airport_name, ad.city
FROM airports_data ad
WHERE ad.city IN (SELECT ad.city
                  FROM airports_data ad
                  GROUP BY ad.city
                  HAVING COUNT(*) > 1);
```


### Найти ближайший вылетающий рейс из Екатеринбурга в Москву, на который еще не завершилась регистрация
 ```postgresql
SELECT f.*
from flights f
         JOIN airports_data ad_dep on ad_dep.airport_code = f.departure_airport
         JOIN airports_data ad_ar on ad_ar.airport_code = f.arrival_airport
WHERE ad_dep.city::json ->> 'ru' = 'Москва'
  and ad_ar.city::json ->> 'ru' = 'Екатеринбург'
  and f.status IN ('On Time', 'Delayed', 'Scheduled')
order by scheduled_departure
limit 1;
```

### Вывести самый дешевый и дорогой билет и стоимость ( в одном результирующем ответе)
 ```postgresql
WITH min_max_t AS (SELECT MIN(amount) AS min_a,
                          MAX(amount) AS max_a
                   FROM ticket_flights)
    (SELECT tf.* FROM ticket_flights tf
                          JOIN min_max_t mmt ON tf.amount = mmt.min_a
     LIMIT 1)
UNION ALL
(SELECT tf.* FROM ticket_flights tf
                      JOIN min_max_t mmt ON tf.amount = mmt.max_a
 LIMIT 1);
```


### Вывести информацию о вылете с наибольшей суммарной стоимостью билетов
 ```postgresql
with sum_t AS (SELECT tf.flight_id, sum(tf.amount) sum_amount
               from ticket_flights tf
               group by tf.flight_id)
SELECT sum_t.sum_amount, f.*
from flights f
         join sum_t on sum_t.flight_id = f.flight_id
where sum_t.sum_amount = (SELECT max(sum_t.sum_amount) from sum_t);
```


### Найти модель самолета, принесшую наибольшую прибыль (наибольшая суммарная стоимость билетов). Вывести код модели, информацию о модели и общую стоимость
 ```postgresql
with sum_t AS (SELECT f2.aircraft_code, sum(tf.amount) sum_amount
               from ticket_flights tf
                        join flights f2 on f2.flight_id = tf.flight_id
                        join bookings.aircrafts_data a on f2.aircraft_code = a.aircraft_code
               group by f2.aircraft_code)
SELECT ad.aircraft_code, ad.model, sum_t.sum_amount
from aircrafts_data ad
         join sum_t on sum_t.aircraft_code = ad.aircraft_code
where sum_t.sum_amount = (SELECT max(sum_t.sum_amount) from sum_t);
```


### Найти самый частый аэропорт назначения для каждой модели самолета. Вывести количество вылетов, информацию о модели самолета, аэропорт назначения, город
 ```postgresql
with count_arrive_flights as (SELECT f.aircraft_code, f.arrival_airport, count(*) flight_count
                              from flights f
                              group by f.aircraft_code, f.arrival_airport)
SELECT arf.flight_count, ad.model, a.airport_name, a.city
from (SELECT c.aircraft_code, max(c.flight_count) max_count from count_arrive_flights c group by c.aircraft_code) m
         join count_arrive_flights arf on arf.aircraft_code = m.aircraft_code and arf.flight_count = m.max_count
         join aircrafts_data ad on arf.aircraft_code = ad.aircraft_code
         join airports_data a on arf.arrival_airport = a.airport_code;
```
