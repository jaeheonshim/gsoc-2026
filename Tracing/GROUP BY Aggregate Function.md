```
#0  Item_sum_min_max::update_field (this=0x7507e803b100)
    at /home/jaeheonshim/gsoc/server/sql/item_sum.cc:3136
#1  0x00005fd565833351 in update_tmptable_sum_func (func_ptr=0x7507e803d290, tmp_table=0x7507e8090580)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:30284
#2  0x00005fd5658290b6 in end_update (join=0x7507e803c5d0, join_tab=0x7507e803e578, end_of_records=false)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:26452
#3  0x00005fd56583e6e6 in AGGR_OP::put_record (this=0x7507e803f558, end_of_records=false)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:33856
#4  0x00005fd565847b6b in AGGR_OP::put_record (this=0x7507e803f558)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.h:1195
#5  0x00005fd565823a23 in sub_select_postjoin_aggr (join=0x7507e803c5d0, join_tab=0x7507e803e578, 
    end_of_records=false) at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:24404
#6  0x00005fd565824ab6 in evaluate_join_record (join=0x7507e803c5d0, join_tab=0x7507e803e100, error=0)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:24953
#7  0x00005fd565824450 in sub_select (join=0x7507e803c5d0, join_tab=0x7507e803e100, end_of_records=false)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:24757
#8  0x00005fd5658232de in do_select (join=0x7507e803c5d0, procedure=0x0)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:24231
#9  0x00005fd5657eba0e in JOIN::exec_inner (this=0x7507e803c5d0)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:5125
#10 0x00005fd5657eaa86 in JOIN::exec (this=0x7507e803c5d0)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:4913
#11 0x00005fd5657ec45f in mysql_select (thd=0x7507e8002758, tables=0x7507e803b2d0, fields=..., 
    conds=0x0, og_num=1, order=0x0, group=0x7507e803bb70, having=0x0, proc_param=0x0, 
    select_options=2164525824, result=0x7507e803c5a8, unit=0x7507e8006e78, select_lex=0x7507e803a978)
    at /home/jaeheonshim/gsoc/server/sql/sql_select.cc:5439
```
## Setup

```sql
CREATE TABLE t (
    id INT AUTO_INCREMENT PRIMARY KEY,
    grp INT,
    val INT,
    name VARCHAR(50)
);

INSERT INTO t (grp, val, name) VALUES
(1, 10, 'a'),
(1, 20, 'b'),
(1, NULL, 'c'),
(2, 5, 'x'),
(2, 15, 'y'),
(2, NULL, NULL),
(3, NULL, NULL);
```

## Trace

Ran
```sql
SELECT grp, MAX(val)
FROM t
GROUP BY grp;
```

As always, execution begins in the `mysql_select` function
### Query Execution
`join->exec()` is called:
```cpp hl:4
if (unlikely(thd->is_error()))
	goto err;

exec_error= join->exec();

if (thd->lex->describe & DESCRIBE_EXTENDED)
{
	select_lex->where= join->conds_history;
	select_lex->having= join->having_history;
}
```

Which goes straight into `exec_inner()`:
```cpp hl:12
int JOIN::exec()
{
  int res;
  DBUG_ASSERT(optimization_state == OPTIMIZATION_DONE);
  DBUG_EXECUTE_IF("show_explain_probe_join_exec_start", 
                  if (dbug_user_var_equals_int(thd, 
                                               "show_explain_probe_select_id", 
                                               select_lex->select_number))
                        dbug_serve_apcs(thd, 1);
                 );
  ANALYZE_START_TRACKING(thd, &explain->time_tracker);
  res= exec_inner();
  ANALYZE_STOP_TRACKING(thd, &explain->time_tracker);

  DBUG_EXECUTE_IF("show_explain_probe_join_exec_end", 
                  if (dbug_user_var_equals_int(thd, 
                                               "show_explain_probe_select_id", 
                                               select_lex->select_number))
                        dbug_serve_apcs(thd, 1);
                 );
  return res;
}
```

The function we're interested in, `do_select`, is called at around line 5125
```cpp hl:8
int JOIN::exec_inner()
{
...
  result->send_result_set_metadata(
                 procedure ? procedure_fields_list : *fields,
                 Protocol::SEND_NUM_ROWS | Protocol::SEND_EOF);

  error= result->view_structure_only() ? false : do_select(this, procedure);
  /* Accumulate the counts from all join iterations of all join parts. */
  thd->ps_report_examined_row_count();

  DBUG_PRINT("counts", ("thd->examined_row_count: %lu",
                        (ulong) thd->get_examined_row_count()));

  DBUG_RETURN(error);
}
```

A `JOIN_TAB` is created, and we run `first_select` passing it as an argument (line 24231)
```cpp hl:14
static int
do_select(JOIN *join, Procedure *procedure)
{
...
    if (top_level_tables)
      join->join_tab[top_level_tables-1].cached_pfs_batch_update=
        join->join_tab[top_level_tables-1].pfs_batch_update();

    JOIN_TAB *join_tab= join->join_tab +
                        (join->tables_list ? join->const_tables : 0);
    if (join->outer_ref_cond && !join->outer_ref_cond->val_bool())
      error= NESTED_LOOP_NO_MORE_ROWS;
    else
      error= join->first_select(join,join_tab,0);
    if (error >= NESTED_LOOP_OK && likely(join->thd->killed != ABORT_QUERY))
      error= join->first_select(join,join_tab,1);
  }
```

First select by default points to `sub_select` inside which is this while loop
```cpp
while (rc == NESTED_LOOP_OK && join->return_tab >= join_tab)
{
if (join_tab->loosescan_match_tab && 
	join_tab->loosescan_match_tab->found_match)
{
  KEY *key= join_tab->table->key_info + join_tab->loosescan_key;
  key_copy(join_tab->loosescan_buf, join_tab->table->record[0], key, 
		   join_tab->loosescan_key_len);
  skip_over= TRUE;
}

error= info->read_record();

if (skip_over && likely(!error))
{
  if (!key_cmp(join_tab->table->key_info[join_tab->loosescan_key].key_part,
			   join_tab->loosescan_buf, join_tab->loosescan_key_len))
  {
	/* 
	  This is the LooseScan action: skip over records with the same key
	  value if we already had a match for them.
	*/
	continue;
  }
  join_tab->loosescan_match_tab->found_match= FALSE;
  skip_over= FALSE;
}

if (join_tab->keep_current_rowid && likely(!error))
  join_tab->table->file->position(join_tab->table->record[0]);

rc= evaluate_join_record(join, join_tab, error);
}
```