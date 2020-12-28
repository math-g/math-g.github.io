---
layout: post
title:  "Using Oracle hint usage report to analyse conflicts between LEADING and INLINE hints"
comments: true
date:   2020-12-28 23:21:58 +0200
---

Oracle 19c came with a new important feature : the hint usage report.

A lot of articles or videos have been produced to explain why a specific hint was not used by the optimizer.  
All this specific knowledge and experience become now obsolete, with Oracle telling you precisely and in plain text why a given hint was ignored.

For me, it was an opportunity to revisit and improve some queries, and precisely confirm some theories about the optimizer.

Yes theories, because, Oracle optimizer being closed source and so complex, working with it mainly consists in empirical observations, like scientists, instead of looking a the source code as we could do as developers.

One particular aspect I inspect here is the specific behaviour of the `LEADING` hint when combined with `INLINE`, that could lead to ignore the `LEADING` hint.  
Because it defines join order, it is a crucial hint and must not be ignored.

This is something that was known before the hint usage report, but sometimes buried in complex queries, problems with this combination could escape one's attention and manage to go to production.
Most of the time the plan remains inchanged as the optimizer still picks the best join order even when the hint is ignored.  
But one day, the plan can suddenly change, and this will of course be always at the worst moment.    
If you must rely on hints to stabilize a plan instead of baselines, the hint usage report is helping us at the development step to avoid this pitfall.

To illustrate this point, consider this very simple query, with two placeholders for the different hints that will be tested :

    WITH table1 AS (
        SELECT 1 AS id, 'a' AS value FROM dual UNION ALL
        SELECT 2 AS id, 'b' AS value FROM dual UNION ALL
        SELECT 3 AS id, 'c' AS value FROM dual UNION ALL
        SELECT 4 AS id, 'd' AS value FROM dual UNION ALL
        SELECT 5 AS id, 'e' AS value FROM dual UNION ALL
        SELECT 6 AS id, 'f' AS value FROM dual UNION ALL
        SELECT 7 AS id, 'g' AS value FROM dual
    ),
    table2 AS (
        SELECT 1 AS id, 'a' AS value FROM dual UNION ALL
        SELECT 2 AS id, 'b' AS value FROM dual UNION ALL
        SELECT 3 AS id, 'c' AS value FROM dual UNION ALL
        SELECT 4 AS id, 'd' AS value FROM dual UNION ALL
        SELECT 5 AS id, 'e' AS value FROM dual
    ),
    table3 AS (
        SELECT 1 AS id, 'a' AS value FROM dual UNION ALL
        SELECT 2 AS id, 'b' AS value FROM dual UNION ALL
        SELECT 3 AS id, 'c' AS value FROM dual
    ),
    view1 AS (
        SELECT /*+ ###FIRST HINT### */
        table1.id, table2.value
        FROM table1 JOIN table2 ON table1.id = table2.id
    )
    SELECT /*+ ###SECOND HINT### */
    view1.id, table3.value
    FROM view1 JOIN table3 ON view1.id = table3.id




#### 1) Combining INLINE for a view with LEADING in another view using it :

After replacements, the query becomes :

    ...
    view1 AS (
        SELECT /*+ LEADING(table1 table2) INLINE */
        table1.id, table2.value
        FROM table1 JOIN table2 ON table1.id = table2.id
    )
    SELECT /*+ LEADING(view1 table3) */
    view1.id, table3.value
    FROM view1 JOIN table3 ON view1.id = table3.id

The resulting plan :
 
    -------------------------------------------------------------------------
    | Id  | Operation        | Name | Rows  | Bytes | Cost (%CPU)| Time     |
    -------------------------------------------------------------------------
    |   0 | SELECT STATEMENT |      |     1 |    12 |    30   (0)| 00:00:01 |
    |*  1 |  HASH JOIN       |      |     1 |    12 |    30   (0)| 00:00:01 |
    |*  2 |   HASH JOIN      |      |     1 |     6 |    24   (0)| 00:00:01 |
    |   3 |    VIEW          |      |     5 |    15 |    10   (0)| 00:00:01 |
    |   4 |     UNION-ALL    |      |       |       |            |          |
    |   5 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   6 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   7 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   8 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   9 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  10 |    VIEW          |      |     7 |    21 |    14   (0)| 00:00:01 |
    |  11 |     UNION-ALL    |      |       |       |            |          |
    |  12 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  13 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  14 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  15 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  16 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  17 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  18 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  19 |   VIEW           |      |     3 |    18 |     6   (0)| 00:00:01 |
    |  20 |    UNION-ALL     |      |       |       |            |          |
    |  21 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    |  22 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    |  23 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    -------------------------------------------------------------------------

The hint usage report :

    Hint Report (identified by operation id / Query Block Name / Object Alias):
    Total hints for statement: 3 (U - Unused (2))
    ---------------------------------------------------------------------------
     
       1 -  SEL$09E891C9
             U -  LEADING(table1 table2) / hint conflicts with another in sibling query block
             U -  LEADING(view1 table3) / hint conflicts with another in sibling query block
               -  INLINE
           

Here we see an interesting behaviour : because the view `view1` is inlined, its hint is also inlined in any "client" view or query.  

So the final query with hints, after inlining, would look like :
    
    SELECT /*+ LEADING(table1 table2) LEADING(view1 table3) */
    view1.id, table3.value
    FROM (
      SELECT table1.id, table2.value
      FROM table1 JOIN table2 ON table1.id = table2.id) view1
    JOIN table3 ON view1.id = table3.id
    
instead of :

    SELECT /*+ LEADING(view1 table3) */
    view1.id, table3.value
    FROM (SELECT /*+ LEADING(table1 table2) */
        table1.id, table2.value
        FROM table1 JOIN table2 ON table1.id = table2.id) view1
    JOIN table3 ON view1.id = table3.id

And that is the hint conflict reported by Oracle.
Because both hints are prefixed with `U`, they are unused (ignored) and we now rely on the optimizer to determine the join order.  
As explained above, this could be sometimes problematic.

Let's try to confirm that it is really the INLINE which cause the conflict.


#### 2) Replacing INLINE with MATERIALIZE 

After replacements, the query becomes :

    ...
    view1 AS (
        SELECT /*+ LEADING(table1 table2) MATERIALIZE */
        table1.id, table2.value
        FROM table1 JOIN table2 ON table1.id = table2.id
    )
    SELECT /*+ LEADING(view1 table3) */
    view1.id, table3.value
    FROM view1 JOIN table3 ON view1.id = table3.id

The resulting plan shows the materialization :

    ----------------------------------------------------------------------------------------------------------------------
    | Id  | Operation                                | Name                      | Rows  | Bytes | Cost (%CPU)| Time     |
    ----------------------------------------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT                         |                           |     1 |     9 |    32   (0)| 00:00:01 |
    |   1 |  TEMP TABLE TRANSFORMATION               |                           |       |       |            |          |
    |   2 |   LOAD AS SELECT (CURSOR DURATION MEMORY)| SYS_TEMP_0FD9DEFDC_90E67D |       |       |            |          |
    |*  3 |    HASH JOIN                             |                           |     1 |     9 |    24   (0)| 00:00:01 |
    |   4 |     VIEW                                 |                           |     7 |    21 |    14   (0)| 00:00:01 |
    |   5 |      UNION-ALL                           |                           |       |       |            |          |
    |   6 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |   7 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |   8 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |   9 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |  10 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |  11 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |  12 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |  13 |     VIEW                                 |                           |     5 |    30 |    10   (0)| 00:00:01 |
    |  14 |      UNION-ALL                           |                           |       |       |            |          |
    |  15 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |  16 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |  17 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |  18 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |  19 |       FAST DUAL                          |                           |     1 |       |     2   (0)| 00:00:01 |
    |* 20 |   HASH JOIN                              |                           |     1 |     9 |     8   (0)| 00:00:01 |
    |  21 |    VIEW                                  |                           |     1 |     3 |     2   (0)| 00:00:01 |
    |  22 |     TABLE ACCESS FULL                    | SYS_TEMP_0FD9DEFDC_90E67D |     1 |     9 |     2   (0)| 00:00:01 |
    |  23 |    VIEW                                  |                           |     3 |    18 |     6   (0)| 00:00:01 |
    |  24 |     UNION-ALL                            |                           |       |       |            |          |
    |  25 |      FAST DUAL                           |                           |     1 |       |     2   (0)| 00:00:01 |
    |  26 |      FAST DUAL                           |                           |     1 |       |     2   (0)| 00:00:01 |
    |  27 |      FAST DUAL                           |                           |     1 |       |     2   (0)| 00:00:01 |
    ----------------------------------------------------------------------------------------------------------------------

And this is the corresponding hint usage report :
	
    Hint Report (identified by operation id / Query Block Name / Object Alias):
    Total hints for statement: 3
    ---------------------------------------------------------------------------
     
       1 -  SEL$174C6AE6
               -  LEADING(view1 table3)
     
       2 -  SEL$904DB506
               -  LEADING(table1 table2)
               -  MATERIALIZE
               
           
Both hints were now used.  
Because we have materialized the first view, the first LEADING hint acts now in its own scope on the view `view1` and will not conflict the the main query with the second LEADING, referencing this view.

Now let's repair the first SQL.
      
#### 3) Removing conflicting hint :
 
To prevent the conflict, we just have to remove the second LEADING hint.
After replacements, the query becomes :
 
     ...
     view1 AS (
        SELECT /*+ LEADING(table1 table2) INLINE */
        table1.id, table2.value
        FROM table1 JOIN table2 ON table1.id = table2.id
    )
    SELECT 
    view1.id, table3.value
    FROM view1 JOIN table3 ON view1.id = table3.id

The plan, similar to the first one but not exactly the same :

     -------------------------------------------------------------------------
    | Id  | Operation        | Name | Rows  | Bytes | Cost (%CPU)| Time     |
    -------------------------------------------------------------------------
    |   0 | SELECT STATEMENT |      |     1 |    12 |    30   (0)| 00:00:01 |
    |*  1 |  HASH JOIN       |      |     1 |    12 |    30   (0)| 00:00:01 |
    |*  2 |   HASH JOIN      |      |     1 |     6 |    24   (0)| 00:00:01 |
    |   3 |    VIEW          |      |     7 |    21 |    14   (0)| 00:00:01 |
    |   4 |     UNION-ALL    |      |       |       |            |          |
    |   5 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   6 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   7 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   8 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   9 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  10 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  11 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  12 |    VIEW          |      |     5 |    15 |    10   (0)| 00:00:01 |
    |  13 |     UNION-ALL    |      |       |       |            |          |
    |  14 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  15 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  16 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  17 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  18 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  19 |   VIEW           |      |     3 |    18 |     6   (0)| 00:00:01 |
    |  20 |    UNION-ALL     |      |       |       |            |          |
    |  21 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    |  22 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    |  23 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    -------------------------------------------------------------------------
    
The corresponding hint usage report :

    Hint Report (identified by operation id / Query Block Name / Object Alias):
    Total hints for statement: 2
    ---------------------------------------------------------------------------
     
       1 -  SEL$09E891C9
               -  INLINE
               -  LEADING(table1 table2)
               
Now there is no more conflict and importantly this time, the join order in the main query is the one we expected.  
Actually, when we were using the two LEADING hints, all that we wanted was to force the main query to start from `view1`, which itself starts from `table1`.  
But removing this second hint, we see that the first LEADING hint is indeed inlined in the main query, forcing it to start from `table1`.   
If we start from `table1`, we start from `view1`, and that was our goal.    
So definitely, the solution here is to suppress the second LEADING hint.

We also remarked that the plan in 1) is different from this one.
Because both LEADING hints were ignored, we relied on the optimizer to define join order and in this case, this differed from the order we wanted to force.  
That is the main point here : when we have to stabilize a plan of a complex query with hints only, we know that the optimizer join order choices if we let him without hints can differ from the one with hints.
And we know that it will not be instantly obvious if for one time after development the optimizer choose the "right" plan.
So we must ensure during development that no hint was ignored, and that is possible from 19c.

Let's make a last observation, replacing the second LEADING with ORDERED.

#### 4) Replacing the second LEADING with ORDERED, modifying the join order

After replacements and modification, the query becomes :

    ...
    view1 AS (
        SELECT /*+ LEADING(table1 table2) INLINE */
        table1.id, table2.value
        FROM table1 JOIN table2 ON table1.id = table2.id
    )
    SELECT /*+ ORDERED */
    view1.id, table3.value
    FROM table3 JOIN view1 ON view1.id = table3.id

Here, `view1` and `table3` were switched to create a conflict with the LEADING order coming from the inlined view.

The plan :

    -------------------------------------------------------------------------
    | Id  | Operation        | Name | Rows  | Bytes | Cost (%CPU)| Time     |
    -------------------------------------------------------------------------
    |   0 | SELECT STATEMENT |      |     1 |    12 |    30   (0)| 00:00:01 |
    |*  1 |  HASH JOIN       |      |     1 |    12 |    30   (0)| 00:00:01 |
    |*  2 |   HASH JOIN      |      |     7 |    63 |    20   (0)| 00:00:01 |
    |   3 |    VIEW          |      |     3 |    18 |     6   (0)| 00:00:01 |
    |   4 |     UNION-ALL    |      |       |       |            |          |
    |   5 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   6 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   7 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |   8 |    VIEW          |      |     7 |    21 |    14   (0)| 00:00:01 |
    |   9 |     UNION-ALL    |      |       |       |            |          |
    |  10 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  11 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  12 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  13 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  14 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  15 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  16 |      FAST DUAL   |      |     1 |       |     2   (0)| 00:00:01 |
    |  17 |   VIEW           |      |     5 |    15 |    10   (0)| 00:00:01 |
    |  18 |    UNION-ALL     |      |       |       |            |          |
    |  19 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    |  20 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    |  21 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    |  22 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    |  23 |     FAST DUAL    |      |     1 |       |     2   (0)| 00:00:01 |
    -------------------------------------------------------------------------

Ane the hint usage report :
	
    Hint Report (identified by operation id / Query Block Name / Object Alias):
    Total hints for statement: 3 (U - Unused (1))
    ---------------------------------------------------------------------------
     
       1 -  SEL$09E891C9
             U -  LEADING(table1 table2) / hint overridden by another in parent query block
               -  INLINE
               -  ORDERED
           
The hint usage report tells us that this time, the inlined LEADING hint was ignored, overriden by the ORDERED one, wich forced a different join order.

Is saw this overriding to be not systematic.  
If, in the main query, we join more tables than `table3` only, and if the ordering corresponds to the inlined LEADING, both LEADING and ORDERED can be kept at the same time.

For me, this illustrate another point which is that some combinations, like these ones, were difficult to be sure about before the hint usage report was available.
