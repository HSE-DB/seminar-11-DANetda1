## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*

    <img width="1484" height="271" alt="image" src="https://github.com/user-attachments/assets/5f6d6306-3894-41ef-8223-d15293a19bc4" />

    
    *Объясните результат:*

    PostgreSQL использует индекс по category, поэтому план идёт через Bitmap Index Scan, а потом Bitmap Heap Scan.
    Так как строк с category = 'A' очень много (примерно половина таблицы), всё равно приходится читать много блоков таблицы (Heap Blocks: exact=8334).
    Recheck Cond появляется потому что после выбора страниц по bitmap условие перепроверяется на строках.
    В итоге запрос не супер быстрый, потому что данных реально вытаскивается очень много.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*

    <img width="866" height="53" alt="image" src="https://github.com/user-attachments/assets/23b9b21e-c7f2-43d3-9704-50984b4b0d2a" />


5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*

    <img width="1491" height="268" alt="image" src="https://github.com/user-attachments/assets/52d82fb9-cb42-4df1-a05f-47b85de663c8" />

    
    *Объясните результат:*

    План остался таким же: Bitmap Index Scan по индексу и потом Bitmap Heap Scan по таблице, потому что условие то же и строк всё равно очень много.
    Но после CLUSTER строки с одинаковой категорией лежат ближе друг к другу, поэтому таблица читается более “кучно”.
    Это видно по Heap Blocks: стало 4167 вместо 8334, то есть нужно прочитать меньше блоков.
    Поэтому общее время выполнения уменьшилось (Execution Time стало меньше).

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*

    До кластеризации запрос выполнялся дольше (Execution Time ~120 ms), после кластеризации — быстрее (~93 ms).
    Основная причина — после CLUSTER строки категории 'A' оказались сгруппированы, и PostgreSQL читает меньше блоков таблицы (8334 → 4167).
    План при этом не изменился (всё ещё bitmap scan), просто уменьшилось количество чтения с диска/страниц, поэтому и ускорение.
