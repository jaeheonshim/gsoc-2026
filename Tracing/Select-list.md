Command executed:
```sql
SELECT (SELECT MIN(c0) FROM t2) < 0 OR true;
```

## Setup
The entry-point for command execution is `mysql_execute_command` in `sql/sql_parse.cc`.

There's a giant switch statement that switches on what the type of the sql command is

```cpp
  switch (lex->sql_command) {

  case SQLCOM_SHOW_EVENTS:
#ifndef HAVE_EVENT_SCHEDULER
    my_error(ER_NOT_SUPPORTED_YET, MYF(0), "embedded server");
    break;
#endif
  case SQLCOM_SHOW_STATUS:
  {
    WSREP_SYNC_WAIT(thd, WSREP_SYNC_WAIT_BEFORE_SHOW);
    execute_show_status(thd, all_tables);

    break;
  }
  case SQLCOM_SHOW_EXPLAIN:
...
```

On line 3953 is `SQLCOM_SELECT` which is where we reach. Then we check table access and run execute_sqlcom_select

```cpp
 if (all_tables)
      res= check_table_access(thd,
                              privileges_requested,
                              all_tables, FALSE, UINT_MAX, FALSE);
    else
      res= check_access(thd, privileges_requested, any_db.str, NULL,NULL,0,0);

    if (!res)
      res= execute_sqlcom_select(thd, all_tables);
```

That function is actually pretty compact so I can put it all here:

```cpp
static bool execute_sqlcom_select(THD *thd, TABLE_LIST *all_tables)
{
  LEX	*lex= thd->lex;
  select_result *result=lex->result;
  bool res;
  /* assign global limit variable if limit is not given */
  {
    SELECT_LEX *param= lex->unit.global_parameters();
    if (!param->limit_params.explicit_limit)
      param->limit_params.select_limit=
        new (thd->mem_root) Item_int(thd,
                                     (ulonglong) thd->variables.select_limit);
  }

  if (!(res= open_and_lock_tables(thd, all_tables, TRUE, 0)))
  {
    if (lex->describe)
    {
      /*
        We always use select_send for EXPLAIN, even if it's an EXPLAIN
        for SELECT ... INTO OUTFILE: a user application should be able
        to prepend EXPLAIN to any query and receive output for it,
        even if the query itself redirects the output.
      */
      if (unlikely(!(result= new (thd->mem_root) select_send(thd))))
        return 1;                               /* purecov: inspected */
      thd->send_explain_fields(result, lex->describe, lex->analyze_stmt);
        
      /*
        This will call optimize() for all parts of query. The query plan is
        printed out below.
      */
      res= mysql_explain_union(thd, &lex->unit, result);
      
      /* Print EXPLAIN only if we don't have an error */
      if (likely(!res))
      {
        /* 
          Do like the original select_describe did: remove OFFSET from the
          top-level LIMIT
        */
        result->remove_offset_limit();
        if (lex->explain_json)
        {
          lex->explain->print_explain_json(result, lex->analyze_stmt);
        }
        else
        {
          lex->explain->print_explain(result, thd->lex->describe,
                                      thd->lex->analyze_stmt);
          if (lex->describe & DESCRIBE_EXTENDED)
          {
            char buff[1024];
            String str(buff,(uint32) sizeof(buff), system_charset_info);
            str.length(0);
            /*
              The warnings system requires input in utf8, @see
              mysqld_show_warnings().
            */
            lex->unit.print(&str, QT_EXPLAIN_EXTENDED);
            push_warning(thd, Sql_condition::WARN_LEVEL_NOTE,
                         ER_YES, str.c_ptr_safe());
          }
        }
      }

      if (res)
        result->abort_result_set();
      else
        result->send_eof();
      delete result;
    }
    else
    {
      Protocol *save_protocol= NULL;
      if (lex->analyze_stmt)
      {
        if (result && result->result_interceptor())
          result->result_interceptor()->disable_my_ok_calls();
        else 
        {
          DBUG_ASSERT(thd->protocol);
          result= new (thd->mem_root) select_send_analyze(thd);
          save_protocol= thd->protocol;
          thd->protocol= new Protocol_discard(thd);
        }
      }
      else
      {
        if (!result && !(result= new (thd->mem_root) select_send(thd)))
          return 1;                               /* purecov: inspected */
      }
      query_cache_store_query(thd, all_tables);
      res= handle_select(thd, lex, result, 0);
      if (result != lex->result)
        delete result;

      if (lex->analyze_stmt)
      {
        if (save_protocol)
        {
          delete thd->protocol;
          thd->protocol= save_protocol;
        }
        if (!res)
	{
          bool extended= thd->lex->describe & DESCRIBE_EXTENDED;
          res= thd->lex->explain->send_explain(thd, extended);
        }
      }
    }
  }
  /*
    Count number of empty select queries.
    is_cursor_execution is used to handle opening of cursor.
    For cursors, Empty_queries will be set when using the cursor.
   */
  if (unlikely(!thd->get_sent_row_count() && !thd->is_cursor_execution() &&
               !(thd->server_status & SERVER_STATUS_RETURNED_ROW) && !res))
    status_var_increment(thd->status_var.empty_queries);
  else
    status_var_add(thd->status_var.rows_sent, thd->get_sent_row_count());

  return res;
}
```

The true path of this if statement (`if (lex->describe)`) executes when you're running an EXPLAIN query. Otherwise, you fall into the else branch, and actually execute the select. In this function we call `mysql_select`:

```cpp
bool handle_select(THD *thd, LEX *lex, select_result *result,
                   ulonglong setup_tables_done_option)
{
  bool res;
  SELECT_LEX *select_lex= lex->first_select_lex();
  DBUG_ENTER("handle_select");
  MYSQL_SELECT_START(thd->query());

  if (select_lex->master_unit()->is_unit_op() ||
      select_lex->master_unit()->fake_select_lex)
    res= mysql_union(thd, lex, result, &lex->unit, setup_tables_done_option);
  else
  {
    SELECT_LEX_UNIT *unit= &lex->unit;
    unit->set_limit(unit->global_parameters());
    /*
      'options' of mysql_select will be set in JOIN, as far as JOIN for
      every PS/SP execution new, we will not need reset this flag if 
      setup_tables_done_option changed for next reexecution
    */
    res= mysql_select(thd,
		      select_lex->table_list.first,
		      select_lex->item_list,
		      select_lex->where,
		      select_lex->order_list.elements +
		      select_lex->group_list.elements,
		      select_lex->order_list.first,
		      select_lex->group_list.first,
		      select_lex->having,
		      lex->proc_list.first,
		      select_lex->options | thd->variables.option_bits |
                      setup_tables_done_option,
		      result, unit, select_lex);
...
```
## mysql_select (`sql/sql_select.cc`)
```cpp
bool
mysql_select(THD *thd, TABLE_LIST *tables, List<Item> &fields, COND *conds,
             uint og_num, ORDER *order, ORDER *group, Item *having,
             ORDER *proc_param, ulonglong select_options, select_result *result,
             SELECT_LEX_UNIT *unit, SELECT_LEX *select_lex)
```

First we see a few conditionals
- `if (!fields.is_empty())` - check if there are any fields referenced by this query. in our case, fields has size 1
- `if (select_lex->join != 0)` - are there joins? in our case, no

We fall into the else branch of the second conditional where we have this
```cpp
 if (!(join= new (thd->mem_root) JOIN(thd, fields, select_options, result)))
	DBUG_RETURN(TRUE);
```
A [[JOIN object]] is created. Then we prepare it:
```cpp
if (!join->prepared &&
	(err= join->prepare(tables, conds, og_num, order, false, group, having,
						proc_param, select_lex, unit)))
{
  goto err;
}
```
