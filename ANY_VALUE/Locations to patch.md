These are the parts of the code we'll need to patch so that when the `ANY_VALUE`  function is used, unaggregated fields in the select list do not throw an error under `ONLY_FULL_GROUP_BY`

Likely the easiest thing to do will be to "trick" `setup_fields` into thinking that a field surrounded by `ANY_VALUE()` is aggregated
- This is basically what MySQL does, when the syntax tree of the query is walked it skips checking functional dependency on GROUP BY when it encounters `ANY_VALUE`
- There are a bunch of edge cases that need to be handled if we make ANY_VALUE an actual aggregate
	- You wouldn't be able to use it in WHERE clause, even though this is valid
	- It would force the query to be aggregated e.g.
		- `SELECT ANY_VALUE(x) FROM t`
		- Should return n rows but with `ANY_VALUE` as aggregate fn it would only return 1
	- etc

## In `Item_field::fix_fields`

```cpp hl:2,9 error:5-6
if (!thd->lex->in_sum_func && !thd->lex->in_any_value)
  select_lex->set_non_agg_field_used(true);
else
{
  if (outer_fixed)
	thd->lex->in_sum_func->outer_fields.push_back(this, thd->mem_root);
  else if (thd->lex->in_sum_func->nest_level !=
	  select->nest_level)
	select_lex->set_non_agg_field_used(true);
}
```

We can mirror behavior of in_sum_func by adding a new in_any_value field to LEX of type `Item_func_any_value *`

In the `Item_func_any_value` we'll keep track of the current nest level of the ANY_VALUE function, like we do in sum funcs.
## Hoisting behavior
Suppose your aggregate function references an outer field. Then there are cases
### Case 1, success: function aggregates at outer level
```sql
SELECT t1.a, 
 (SELECT SUM(t1.b) FROM t2)
FROM t1
GROUP BY t1.a;
```
Here you can aggregate at level 1 since that's the innermost level where all of the column references inside the aggregate function are contained. So `SUM(t1.b)` is computed for every group (`GROUP BY t1.a`) and passed into the inner query as a scalar
### Case 2, fail: function aggregates at inner level, leaving a field behind
```sql
SELECT t1.a,
	(SELECT ANY_VALUE(t1.b + t2.x) FROM t2)
FROM t1
GROUP BY t1.a;
```
`t1.b` can be aggregated at level 1 but `t2.x` must be aggregated at level 2. So now we're inside level 2 where t1.b is being accessed from the outer query. But the outer query is grouped by t1.a! **So which t1.b within each group are we trying to get? We don't know**

## Handling hoisting
This is handled by adding all outer fields within a sum function to the outer_fields list. Then, inside `Item_sum::check_sum_func` you have this code

```cpp
 List_iterator<Item_field> of(outer_fields);
while ((field= of++))
{
  SELECT_LEX *sel= field->field->table->pos_in_table_list->select_lex;
  if (sel->nest_level < aggr_level)
  {
	if (in_sum_func)
	{
	  in_sum_func->outer_fields.push_back(field, thd->mem_root);
	}
	else
	  sel->set_non_agg_field_used(true);
  }
  if (sel->nest_level > aggr_level &&
	  (sel->agg_func_used()) &&
	  !sel->group_list.elements)
  {
	my_message(ER_MIX_OF_GROUP_FUNC_AND_FIELDS,
			   ER_THD(thd, ER_MIX_OF_GROUP_FUNC_AND_FIELDS), MYF(0));
	return TRUE;
  }
}
```

Now when the aggregation level is known (available in `aggr_level`) we can go through each field and if:
- `sel->nest_level < aggr_level` then the field is outside where the current sum func is aggregated. Either it's aggregated by a different, outer sum func, or we set non agg field used
- `sel->nest_level > aggr_level` TODO

Need to investigate this further in MySQL for `ANY_VALUE`

- Be careful with nesting levels when using them inside a view