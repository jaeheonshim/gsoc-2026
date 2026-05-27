## Trying to test selecting attr not in GROUP BY with ONLY_FULL_GROUP_BY on
I want to see how the program detects and surfaces the `ONLY_FULL_GROUP_BY` error. This will give insights into what corner cases we need to cover and how `ANY_VALUE` will bypass this error

### Executed SQL
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

### Execution Trace
- `mysql_execute_command` (`sql/sql_parse.cc`)
- `execute_sqlcom_select` (`sql/sql_parse.cc`)
- `handle_select` (`sql/sql_select.cc`)
- `mysql_select` (`sql/sql_select.cc`)
- `JOIN::prepare` (`sql/sql_select.cc`)
- `setup_fields` (`sql/sql_base.cc`)
- `Item::fix_fields_if_needed_for_scalar` (`sql/item.h`)
- `Item::fix_fields_if_needed` (`sql/item.h`)
- `Item_field::fix_fields` (`sql/item.cc`)
	- This function handles name resolution for columns. At the very end, we have a few lines of code like this that is run if `MODE_ONLY_FULL_GROUP_BY`
```cpp hl:2,9
if (!thd->lex->in_sum_func)
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

This is the important part for how the only full group by error is triggered. Some logic to unpack here.
 - First, if the current field isn't in an aggregate function at all, then we're definitely not using an aggregated column. So we mark it as true
 - Otherwise, if it is in an aggregate function like `SUM`, then you still don't know if the field is aggregated or not
	 - If the field resolves against an outer query block (`outer_fixed` is true), then it's _possible_ that this field gets hoisted and is aggregated in an outer query block after all. But not for sure. Example:
```sql
SELECT t1.a,
 (SELECT SUM(t1.b) FROM t2)
FROM t1
GROUP BY t1.a;
```

 - Otherwise, we know the field resolves around the current `select`'s query block. So if the agg function's nest level is not equal to the nest level of the current select, then you're not actually aggregating the field within the same select. Example:
```sql
SELECT t1.a, SUM(t1.b + (SELECT t2.c FROM t2 WHERE t2.id = t1.id))
FROM t1
GROUP BY t1.a;
```
`t2.c` is inside a sum but it's not getting aggregated by it!

This is super important as it marks each field with whether it's being aggregated at its level or not. If its not, and we see it used in a `GROUP BY` clause, then that's a violation of ` ONLY_FULL_GROUP_BY` as per MySQL docs:

> MySQL implements detection of functional dependence. If the [`ONLY_FULL_GROUP_BY`](https://dev.mysql.com/doc/refman/9.7/en/sql-mode.html#sqlmode_only_full_group_by) SQL mode is enabled (which it is by default), MySQL rejects queries for which the select list, `HAVING` condition, or `ORDER BY` list refer to nonaggregated columns that are neither named in the `GROUP BY` clause nor are functionally dependent on them.

Now we finish out `setup_fields` and go into the next function. To foreshadow, this function returns a non successful code!

- `setup_without_group` (`sql/sql_select.cc`) (fails here!!!)
	- Stepping through this we can see `res=1` after `select_group` so let's step in there
- `setup_group` (`sql/sql_select.cc`)
	- Looks like the same struct that is used for order by is used for the group by clause lol
	- There's an obvious block here that handles the `ONLY_FULL_GROUP_BY` rejection. Copying a useful comment from the code here for reference
	- `First we check an expression from the select list against the GROUP BY list. If it's found there then it's ok. It's also ok if this expression is a constant or an aggregate function. Otherwise we scan the list of non-aggregated fields and if we'll find at least one field reference that belongs to this expression and doesn't occur in the GROUP BY list we throw an error. If there are no fields in the created list for a select list expression this means that all fields in it are used under aggregate functions.`
	- The while loop is a classic two pointers technique
		- You have the pointer to every expression in the SELECT-list, this is `li`. You also have the pointer to all of the non aggregated fields we got in the previous step, this is `naf_it`. `naf_it` stays behind `it`
		- You walk through every item in the select list, sequentially, and if it falls through this if statement
```cpp
  if (item->type() != Item::SUM_FUNC_ITEM &&
	  item->marker != MARKER_UNDEF_POS &&
	  !item->const_item() &&
	  !(item->real_item()->type() == Item::FIELD_ITEM &&
		item->used_tables() & OUTER_REF_TABLE_BIT))
  {
```
Then we walk through every field in the non agg fields list that corresponds to that expression we are iterating over. This is done by advancing the `naf_it` pointer into this window
```cpp
if (field->marker < cur_pos_in_select_list)
	goto next_field;
/* Found a field from the next expression. */
if (field->marker > cur_pos_in_select_list)
	break;
```
When we're going through all of the non agg fields in that expression, if the field is not in the group by, we raise the error
```cpp
for (ord= order; ord; ord= ord->next)
	if ((*ord->item)->eq((Item*)field, 0))
	  goto next_field;
/*
TODO: change ER_WRONG_FIELD_WITH_GROUP to more detailed
ER_NON_GROUPING_FIELD_USED
*/
my_error(ER_WRONG_FIELD_WITH_GROUP, MYF(0), field->full_name());
return 1;
```
And that's it!

## Notes
- `thd->lex->in_sum_func` is set in `Item_sum::init_sum_func_check`
	- So if we make our `ANY_VALUE` a descendant of `Item_sum`, all of the aggregation checking should come for free