Now let's run an actual aggregation query and see how the parser does not raise an `ONLY_FULL_GROUP_BY` error.

## Executed SQL
```sql
CREATE TABLE t (
    id INT AUTO_INCREMENT PRIMARY KEY,
    grp INT,
    val INT
);

INSERT INTO t (grp, val) VALUES
(1, 10),
(1, 20),
(1, NULL),
(2, 5),
(2, 15),
(2, NULL),
(3, NULL);

SET SESSION sql_mode = (SELECT CONCAT(@@sql_mode, ',ONLY_FULL_GROUP_BY'));

SELECT grp, val FROM t GROUP BY grp;
```