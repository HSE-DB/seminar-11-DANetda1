# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*

   <img width="1286" height="231" alt="image" src="https://github.com/user-attachments/assets/a0d575c1-2a5c-4b5c-ab45-17059eea11a7" />

   
   *Объясните результат:*

   В запросе используется BRIN-индекс по колонке category, поэтому PostgreSQL сначала обращается к индексу, а не просматривает всю таблицу.
   Так как BRIN-индекс работает по диапазонам страниц, выполняется bitmap scan с последующей перепроверкой условий.
   В результате строки с NULL значением не найдены, поэтому запрос вернул 0 строк.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*

   <img width="1331" height="338" alt="image" src="https://github.com/user-attachments/assets/af4c3063-a88b-4771-aa42-c923b9aeadb3" />

   
   *Объясните результат (обратите внимание на bitmap scan):*

   PostgreSQL использует BRIN-индекс по category, поэтому получился Bitmap Index Scan → Bitmap Heap Scan: сначала отмечаются подходящие страницы, потом читается таблица.
   Но BRIN даёт “грубое” попадание по страницам, поэтому пошла перепроверка (Rows Removed by Index Recheck: 150000) и всё отфильтровалось — в итоге строк нет.
   Условие по author = 'SYSTEM' применилось уже после чтения строк (как Filter), то есть индекс по автору тут не использовался.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*

   <img width="1328" height="336" alt="image" src="https://github.com/user-attachments/assets/35404240-de80-4b4d-8bf3-5610cd5e226a" />

   
   *Объясните результат:*

   Для DISTINCT category PostgreSQL сделал Seq Scan по всей таблице, потому что нужно посмотреть все строки и собрать уникальные значения.
   Уникальные категории он собрал через HashAggregate, а потом отдельно сделал Sort, чтобы выдать их в порядке ORDER BY category.
   В итоге категорий получилось мало (6 штук), но чтение таблицы всё равно полное, поэтому основное время уходит на seq scan + агрегацию.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*

   <img width="1328" height="237" alt="image" src="https://github.com/user-attachments/assets/418f69af-a30d-42ed-95be-7dd29157fb11" />

   
   *Объясните результат:*

   Здесь PostgreSQL сделал Seq Scan по всей таблице и просто отфильтровал строки по author LIKE 'S%'.
   По плану видно, что подходящих строк не нашлось (rows=0), поэтому все 150000 строк “удалены фильтром”.
   Индекс по author не помог, потому что BRIN не очень подходит для такого шаблона по строке, и планировщик выбрал обычное сканирование.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*

   <img width="1328" height="235" alt="image" src="https://github.com/user-attachments/assets/83d144d0-b9da-413f-94b5-8b49dc7a9b6d" />

   
   *Объясните результат:*

   Несмотря на созданный индекс по LOWER(title), PostgreSQL всё равно выполнил Seq Scan по всей таблице.
   В плане видно, что условие проверяется как фильтр, и почти все строки были отброшены (Rows Removed by Filter: 149999).
   Планировщик решил, что для такого запроса использование индекса невыгодно, так как подходящих строк очень мало и дешевле пройти таблицу целиком.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*

   <img width="1362" height="306" alt="image" src="https://github.com/user-attachments/assets/c0472d61-71c3-49e3-bac5-01b9308f2673" />

   
   *Объясните результат:*

   После создания составного BRIN-индекса (category, author) запрос стал использовать его сразу по двум условиям, поэтому снова видим Bitmap Index Scan → Bitmap Heap Scan.
   Строк всё равно не нашлось, но перепроверять пришлось уже меньше (Rows Removed by Index Recheck: 8804), и блоков прочитано меньше (lossy=72).
   Поэтому запрос выполнился заметно быстрее, чем в пункте 7 (Execution Time примерно 1.1 ms вместо ~14.5 ms).
