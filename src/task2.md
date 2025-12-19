## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*

    <img width="1256" height="272" alt="image" src="https://github.com/user-attachments/assets/3b3fc241-b167-4dd5-8564-65682d79b0b1" />

    
    *Объясните результат:*

    Запрос использует полнотекстовый GIN-индекс, поэтому сначала идёт Bitmap Index Scan, а потом Bitmap Heap Scan — из индекса берутся подходящие строки и читается таблица только по ним.
    Есть Recheck Cond, потому что PostgreSQL перепроверяет условие на реальных данных после чтения строк.
    В итоге нашлась 1 книга со словом expert, поэтому запрос отработал очень быстро.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*

     <img width="1257" height="167" alt="image" src="https://github.com/user-attachments/assets/d71b8b77-c826-46ed-aad0-8a58a811eca0" />

     
     *Объясните результат:*

     Здесь PostgreSQL сделал Index Scan по первичному ключу t_lookup_pk, потому что поиск идёт по точному значению item_key.
     Поэтому читается сразу нужная строка через индекс, без полного обхода таблицы.
     Из-за этого время выполнения маленькое.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*

     <img width="1431" height="170" alt="image" src="https://github.com/user-attachments/assets/8fd071fa-ad99-4aaf-bde7-84321b2b3946" />

     
     *Объясните результат:*

     Запрос также выполняется через Index Scan по первичному ключу, поэтому PostgreSQL сразу находит нужную строку.
    Так как таблица кластеризована по этому ключу, данные лежат упорядоченно на диске.
    В результате поиск выполняется эффективно и без полного сканирования таблицы.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*

     <img width="1284" height="168" alt="image" src="https://github.com/user-attachments/assets/cc23466a-86ab-4c5d-b946-b70523e6b0a2" />

     
     *Объясните результат:*

     PostgreSQL использует Index Scan по индексу t_lookup_value_idx, так как поиск выполняется по колонке с индексом.
     По индексу быстро проверяется наличие значения T_BOOKS, поэтому полный просмотр таблицы не требуется.
     Подходящих строк найдено не было, но за счёт использования индекса запрос выполнился быстро.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*

     <img width="1482" height="169" alt="image" src="https://github.com/user-attachments/assets/cb3a722c-73da-41f4-b2b2-1e5756cf6b16" />

     
     *Объясните результат:*

     Запрос выполняется через Index Scan по индексу t_lookup_clustered_value_idx, так как поиск идёт по проиндексированной колонке item_value.
     PostgreSQL быстро проверяет индекс, не выполняя полный просмотр таблицы.
     Подходящих строк не найдено, но запрос всё равно отработал быстро за счёт использования индекса.
     Кластеризация таблицы в данном случае почти не влияет, так как поиск идёт напрямую через индекс.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*

     В обоих случаях поиск по значению выполняется с использованием индекса, поэтому полный просмотр таблиц не происходит.
     Разница во времени выполнения между обычной и кластеризованной таблицами минимальна.
     Это связано с тем, что при поиске по индексу PostgreSQL сразу находит нужные строки, и физический порядок данных в таблице почти не влияет.
     Кластеризация даёт больший эффект при последовательном чтении данных, а не при точечном поиске через индекс.
