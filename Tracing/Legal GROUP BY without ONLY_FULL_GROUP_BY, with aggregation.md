## Executed

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

SET GLOBAL sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));

SELECT grp, val FROM t GROUP BY grp;
```
## Execution Trace
- mysql_execute_command
- execute_sqlcom_select
- handle_select
- mysql_select
- JOIN::prepare
	- setup_without_group (`sql/sql_select.cc`)
		- This time, this succeeds since ONLY_FULL_GROUP_BY is not set. So we continue processing the query
- JOIN::exec
	- exec_inner
		- do_select
			- Calls `join->first_select`, which leads to `sub_select`
			- `sub_select` (`sql/sql_select.cc`)
				- There's a big while loop at line 24601 which seems like it's responsible for reading all the rows from the table
				- Note: use`p dbug_print_table_row(join_tab->table)` in gdb to print out the current row
				- `evaluate_join_record`
					- The last `next_select` calls into `sub_select_postjoin_aggr`
					- `sub_select_postjoin_aggr`
						- The comment says: "Accumulate rows of the result of an aggregation operation in a tmp table"
						- It calls `aggr->put_record`
						- `AGGR_OP::put_record`
							- The `join_tab` attached to `AGGR_OP` is a temp table with a single key named "group_key", with the fields being the columns in the GROUP BY clause
							- It calls write_func, which in this case is `end_write`. This just writes into the temp table the row
							- On the second time we get here with a row in the same group, we fall into this branch `if (likely(!table->file->is_fatal_error(error, HA_CHECK_DUP))) goto end;`