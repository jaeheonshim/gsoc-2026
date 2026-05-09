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

SELECT ANY_VALUE(val), grp FROM t GROUP BY grp;
```
## EXPLAIN
```
| -> Table scan on <temporary>  (cost=2.93..5.15 rows=7)
    -> Temporary table with deduplication  (cost=2.56..2.56 rows=7)
        -> Table scan on t  (cost=0.95 rows=7)
```
## Execution Trace
- `mysql_execute_command` (`sql/sql_parse.cc`)
- `Sql_cmd_dml::execute` (`sql/sql_select.cc`)
- `Sql_cmd_dml::execute_inner` (`sql/sql_select.cc`)
- `Query_expression::execute` (`sql/sql_union.cc`)
- `Query_expression::ExecuteIteratorQuery` (`sql/sql_union.cc`)
- `MaterializeIterator::DoInit` (`sql/iterators/composite_iterators.cc`)
- `MaterializeIterator::MaterializeOperand` (`sql/iterators/composite_iterators.cc`)
	- At this point we enter into an infinite while loop. First, we call `read_next_row` which reads the next row returned by the storage engine and applies the functions to it (including `ANY_VALUE`)
	- `MaterializeIterator::read_next_row` (`sql/iterators/composite_iterators.cc`)
		- `copy_funcs` (`sql/sql_executor.cc`)
		- `Item::save_in_field_no_error_check`
		- `Item::save_in_field_inner`
		- `Item_func_numhybrid::val_int'
		- `Item_func_coalesce::int_op`
	- Then, we call `handler::ha_write_row` which eventually calls down to `temptable::Table::insert`. Since the members of the`GROUP BY` form a composite key for this temporary table, the insert fails if a row already exists

When there's an actual aggregate function in the SELECT-list, for example:
```sql
SELECT ANY_VALUE(val), grp, COUNT(*) FROM t GROUP BY grp;
```
The part that happens inside MaterializeOperand is different. See execution trace:

```
| -> Table scan on <temporary>
    -> Aggregate using temporary table
        -> Table scan on t  (cost=0.95 rows=7)
```

But the way `ANY_VALUE` operates is the same. It's just the Item_func_coalesce function with `aggregate_check_group` overridden so that during query parsing, the `ONLY_FULL_GROUP_BY` option does not throw an error.