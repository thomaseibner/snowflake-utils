# snowflake-utils

A few simple utilities to make debugging/tuning easier in Snowflake. 

# debug child session

Retrieving the underlying SQL that a stored procedure executes isn't simple in the classic UI or Snowsight. This procedure takes a query_id and retrieves the underlying sql statements in the session executed during the duration the procedure ran. 

```SQL
CALL DEBUG_CHILD_SESSION('000000-0000-0000-000000000001');
```

|PARENT_QUERY_ID|PARENT_USER_NAME|PARENT_ROLE_NAME|PARENT_QUERY_TEXT|QUERY_ID|QUERY_TEXT|...|
|---|---|---|---|---|---|---|
|000000-0000-0000-000000000001|THOMASEIBNER|ACCOUNTADMIN|CALL DEBUG_CHILD_SESSION('000000-0000-0000-000000000001');|000000-0000-0000-000000000001|CALL DEBUG_CHILD_SESSION('000000-0000-0000-000000000001');|... rest of query_history columns|
|000000-0000-0000-000000000001|THOMASEIBNER|ACCOUNTADMIN|CALL DEBUG_CHILD_SESSION('000000-0000-0000-000000000001');|000000-0000-0000-000000000002|"(WITH running_sessions as (..  order by qh.start_time asc)"|... rest of query_history columns|

At this point you can try to narrow down what is causing your performance issue in the stored procedure. Usually the Snowsight Query Details statistics helps narrow down issues by query_type, total_elapsed_time, bytes_scanned/percentage_scannned_from_cache, bytes_written, rows_produced, etc. 

It needs to be run as a role that has access to `snowflake.account_usage` and for performance reasons the proc only looks back 7 days worth of query_history. I've validated it works both with SQL and JavaScript based stored procedures.

