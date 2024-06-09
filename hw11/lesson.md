> Занятие 15  
Работа с индексами и оптимизация запросов.

Для работы будет использована средняя демо-база авиаперелетов.

---
Упражнение 1 
--- 

Создание обычного индекса. Вкачестве поля выбрано: **bookings.tickets.passenger_id** 
(прим. основание - быстрый поиск билета по номеру документа).

```sql
explain analyze select * from bookings.tickets t where t.passenger_id in ('7460 649076', '2498 327600');
QUERY PLAN
------------------------------------------------------------------------------------------------------------------------+
Gather  (cost=1000.00..19238.28 rows=2 width=104) (actual time=31.352..115.789 rows=2 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on tickets t  (cost=0.00..18238.08 rows=1 width=104) (actual time=55.063..88.739 rows=1 loops=3)
        Filter: ((passenger_id)::text = ANY ('{"7460 649076","2498 327600"}'::text[]))
        Rows Removed by Filter: 276356
Planning Time: 0.399 ms
Execution Time: 115.814 ms

CREATE INDEX tickets_passenger_id_idx ON bookings.tickets (passenger_id);

QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------+
Index Scan using tickets_passenger_id_idx on tickets t  (cost=0.42..16.88 rows=2 width=104) (actual time=2.561..2.754 rows=2 loops=1)
  Index Cond: ((passenger_id)::text = ANY ('{"7460 649076","2498 327600"}'::text[]))
Planning Time: 0.178 ms
Execution Time: 2.769 ms
```

Создание индекса для полнотекстового поиска. Вкачестве поля выбрано: **bookings.tickets.passenger_name** 
(прим. основание - быстрый поиск билета по фамилии и имени).

```sql
explain analyze select * from bookings.tickets t where t.passenger_name = 'NATALYA KUZMINA';
QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------+
Gather  (cost=1000.00..23053.58 rows=75 width=136) (actual time=1.399..49.570 rows=251 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on tickets t  (cost=0.00..22046.08 rows=31 width=136) (actual time=0.663..40.930 rows=84 loops=3)
        Filter: (passenger_name = 'NATALYA KUZMINA'::text)
        Rows Removed by Filter: 276273
Planning Time: 0.189 ms
Execution Time: 49.610 ms

ALTER TABLE bookings.tickets
    ADD COLUMN passenger_name_tsv tsvector
               GENERATED ALWAYS AS (to_tsvector('english', coalesce(passenger_name, ''))) STORED;

CREATE INDEX tickets_passenger_name_tsv_idx ON bookings.tickets USING GIN (passenger_name_tsv);

explain analyze select * from bookings.tickets t where t.passenger_name_tsv @@ to_tsquery('NATALYA & KUZMINA');
QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------+
Bitmap Heap Scan on tickets t  (cost=30.09..117.43 rows=21 width=136) (actual time=0.600..1.357 rows=251 loops=1)
  Recheck Cond: (passenger_name_tsv @@ to_tsquery('NATALYA & KUZMINA'::text))
  Heap Blocks: exact=248
  ->  Bitmap Index Scan on tickets_passenger_name_tsv_idx  (cost=0.00..30.08 rows=21 width=0) (actual time=0.573..0.573 rows=251 loops=1)
        Index Cond: (passenger_name_tsv @@ to_tsquery('NATALYA & KUZMINA'::text))
Planning Time: 0.783 ms
Execution Time: 1.382 ms
```

Создание функционального индекса для поиска. Вкачестве поля выбрано: **bookings.tickets.contact_data** 
(прим. основание - быстрый поиск билета по номеру телефона, а исходное поле в формате *json*).

```sql
explain analyze select * from bookings.tickets ad where (contact_data->>'phone') = '+70981694300';
QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------+
Gather  (cost=1000.00..24324.19 rows=4145 width=136) (actual time=106.602..111.236 rows=1 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on tickets ad  (cost=0.00..22909.69 rows=1727 width=136) (actual time=100.500..101.127 rows=0 loops=3)
        Filter: ((contact_data ->> 'phone'::text) = '+70981694300'::text)
        Rows Removed by Filter: 276357
Planning Time: 0.075 ms
Execution Time: 111.249 ms

CREATE INDEX tickets_contact_data_phone_idx ON bookings.tickets ((contact_data->>'phone'));

explain analyze select * from bookings.tickets ad where (contact_data->>'phone') = '+70981694300';
QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------+
Bitmap Heap Scan on tickets ad  (cost=96.55..9911.03 rows=4145 width=136) (actual time=0.211..0.212 rows=1 loops=1)
  Recheck Cond: ((contact_data ->> 'phone'::text) = '+70981694300'::text)
  Heap Blocks: exact=1
  ->  Bitmap Index Scan on tickets_contact_data_phone_idx  (cost=0.00..95.51 rows=4145 width=0) (actual time=0.204..0.204 rows=1 loops=1)
        Index Cond: ((contact_data ->> 'phone'::text) = '+70981694300'::text)
Planning Time: 0.551 ms
Execution Time: 0.230 ms
```

Создание составного индекса. Вкачестве поля выбрано: **bookings.ticket_flights.flight_id** и **bookings.ticket_flights.fare_conditions**
(прим. основание - формирование списка билетов на конкретный рейс и классом обслуживания).

```sql
explain analyze select * from bookings.ticket_flights ad where (flight_id, fare_conditions) = (33962, 'Business');
QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------+
Gather  (cost=1000.00..35472.99 rows=9 width=32) (actual time=21.370..96.756 rows=9 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on ticket_flights ad  (cost=0.00..34472.09 rows=4 width=32) (actual time=36.968..85.002 rows=3 loops=3)
        Filter: ((flight_id = 33962) AND ((fare_conditions)::text = 'Business'::text))
        Rows Removed by Filter: 786775
Planning Time: 0.090 ms
Execution Time: 96.778 ms

CREATE INDEX ticket_flights_flight_id_fare_conditions_idx ON bookings.ticket_flights (flight_id,fare_conditions);

explain analyze select * from bookings.ticket_flights ad where (flight_id, fare_conditions) = (33962, 'Business');
QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------+
Index Scan using ticket_flights_flight_id_fare_conditions_idx on ticket_flights ad  (cost=0.43..39.79 rows=9 width=32) (actual time=0.059..0.089 rows=9 loops=1)
  Index Cond: ((flight_id = 33962) AND ((fare_conditions)::text = 'Business'::text))
Planning Time: 0.163 ms
Execution Time: 0.101 ms
```