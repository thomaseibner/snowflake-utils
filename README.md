# snowflake-utils

A few simple utilities to make debugging/tuning easier in Snowflake. 

# Nested query debugging 

Both GET\_IS\_NESTED\_QUERY and GET_AU_NESTED_QUERY are wrappers meant to make it easier to debug stored procedures. Snowflake's UI doesn't natively support singling out nested queries. You can look at a session's query history but you can't easily filter by start and end time of the parent query call. With these stored procedures you can easily see the underlying queries and do sorting/filtering in a worksheet in Snowsight or the classic UI. Armed with only the relevant queries you can use the Snowsight 'Query Details' pane to narrow down where your performance issues are.

Choosing between the two versions depends on what you have access to in Snowflake. The information\_schema version requires less privileges, but has some limitations in that it can only give you data back from up to the last 7 days, but because of the way you have to query information_schema you may only be able to look through the 10,000 most recent queries. If you only look for a specific user it is possible to change the query to look at the query\_history\_by\_user view instead of query\_history. The account\_usage version can go back further in time, but requires more privileges. For performance purposes the account\_usage version only looks back to 14 days worth of query history. 

# GET_IS_NESTED_QUERY

Retrieve the underlying SQL that a stored procedure executes - information_schema version. 

Needs to be run as a role that has access to [snowflake.information_schema.query_history](https://docs.snowflake.com/en/sql-reference/functions/query_history). If you only have access to a local database as your user you will need to change it to `table(information\_schema.query\_history())` and have a database selected in your current context. Using it is simple, just pass in the query\_id of the parent query you want to debug:

```SQL
call GET_IS_NESTED_QUERY('000000-0000-0000-000000000001');
```
|PARENT_QUERY_ID|PARENT_SESSION_ID|PARENT_START_TIME|PARENT_END_TIME|PARENT_ELAPSED_TIME|PARENT_USER_NAME|PARENT_ROLE_NAME|PARENT_QUERY_TEXT|PARENT_EXECUTION_STATUS|PARENT_ERROR_CODE|PARENT_ERROR_MESSAGE|QUERY_ID|QUERY_TEXT|...|
|---|---|---|---|---|---|---|---|---|---|---|
|000000-0000-0000-000000000001|00000000000000101|2023-09-16 15:03:23.416 +0000|2023-09-16 15:03:33.216 +0000|9,800|THOMASEIBNER|PRD_ADM_FR|call GET_IS_NESTED_QUERY('000000-0000-0000-000000000001');|SUCCESS|NULL|NULL|000000-0000-0000-000000000001|call GET_IS_NESTED_QUERY('000000-0000-0000-000000000001');|... rest of information_schema.query_history columns|
|000000-0000-0000-000000000001|00000000000000101|2023-09-16 15:03:23.416 +0000|2023-09-16 15:03:33.216 +0000|9,800|THOMASEIBNER|PRD_ADM_FR|call GET_IS_NESTED_QUERY('000000-0000-0000-000000000001');|SUCCESS|NULL|NULL|000000-0000-0000-000000000002|WITH sessions as (select qh.query_id as parent_query_id, |... rest of information_schema.query_history columns|
|000000-0000-0000-000000000001|00000000000000101|2023-09-16 15:03:23.416 +0000|2023-09-16 15:03:33.216 +0000|9,800|THOMASEIBNER|PRD_ADM_FR|call GET_IS_NESTED_QUERY('000000-0000-0000-000000000001');|SUCCESS|NULL|NULL|000000-0000-0000-000000000003||... rest of information_schema.query_history columns|

There is a lot of redundant information in the output, but that is by design if you want to pull it into a streamlit app or similar and still preserve the ability to show the details of the parent query without having to parse the results and find the parent query. 

If there are no rows returned by the queries the procedure will return an error. 

# GET_AU_NESTED_QUERY

Reterieve the underlying SQL that a stored procedure executes - account_usage version.

Needs to run as a role that has access to [snowflake.account_usage.query_history](https://docs.snowflake.com/en/sql-reference/account-usage/query_history.html). If you do not have access to account\_usage you will need to change to a role that does. As with the information\_schema version calling the stored proc is simple and only takes an argument of a query_id:

```SQL
call GET_AU_NESTED_QUERY('000000-0000-0000-000000000001');
```
|PARENT_QUERY_ID|PARENT_SESSION_ID|PARENT_START_TIME|PARENT_END_TIME|PARENT_ELAPSED_TIME|PARENT_USER_NAME|PARENT_ROLE_NAME|PARENT_QUERY_TEXT|PARENT_EXECUTION_STATUS|PARENT_ERROR_CODE|PARENT_ERROR_MESSAGE|QUERY_ID|QUERY_TEXT|...|
|---|---|---|---|---|---|---|---|---|---|---|
|000000-0000-0000-000000000001|00000000000000101|2023-09-16 14:49:33.746 +0000|2023-09-16 14:50:01.856 +0000|28,110|THOMASEIBNER|PRD_ADM_FR|call GET_AU_NESTED_QUERY('000000-0000-0000-000000000001');|SUCCESS|NULL|NULL|000000-0000-0000-000000000001|call GET_IS_NESTED_QUERY('000000-0000-0000-000000000001');|... rest of snowflake.account_usage.query_history columns|
|000000-0000-0000-000000000001|00000000000000101|2023-09-16 14:49:33.746 +0000|2023-09-16 14:50:01.856 +0000|28,110|THOMASEIBNER|PRD_ADM_FR|call GET_AU_NESTED_QUERY('000000-0000-0000-000000000001');|SUCCESS|NULL|NULL|000000-0000-0000-000000000002|WITH sessions as (select qh.query_id as parent_query_id, |... rest of snowflake.account_usage.query_history columns|

If there are no rows returned by the queries the procedure will return an empty table. This is different from the GET\_IS\_NESTED\_QUERY version.

# debug child session (deprecated)

Retrieving the underlying SQL that a stored procedure executes isn't simple in the classic UI or Snowsight. This procedure takes a query_id and retrieves the underlying sql statements in the session executed during the duration the procedure ran. 

```SQL
CALL DEBUG_CHILD_SESSION('000000-0000-0000-000000000001');
```
|PARENT_QUERY_ID|PARENT_USER_NAME|PARENT_ROLE_NAME|PARENT_EXECUTION_STATUS|PARENT_ERROR_CODE|PARENT_ERROR_MESSAGE|PARENT_QUERY_TEXT|QUERY_ID|QUERY_TEXT|...|
|---|---|---|---|---|---|---|
|000000-0000-0000-000000000001|THOMASEIBNER|ACCOUNTADMIN|SUCCESS|NULL|NULL|CALL DEBUG_CHILD_SESSION('000000-0000-0000-000000000001');|000000-0000-0000-000000000001|CALL DEBUG_CHILD_SESSION('000000-0000-0000-000000000001');|... rest of query_history columns|
|000000-0000-0000-000000000001|THOMASEIBNER|ACCOUNTADMIN|SUCCESS|NULL|NULL|CALL DEBUG_CHILD_SESSION('000000-0000-0000-000000000001');|000000-0000-0000-000000000002|"(WITH running_sessions as (..  order by qh.start_time asc)"|... rest of query_history columns|

At this point you can try to narrow down what is causing your performance issue in the stored procedure. Usually the Snowsight Query Details statistics helps narrow down issues by query_type, total_elapsed_time, bytes_scanned/percentage_scannned_from_cache, bytes_written, rows_produced, etc. 

It needs to be run as a role that has access to `snowflake.account_usage` and for performance reasons the proc only looks back 7 days worth of query_history. I've validated it works both with SQL and JavaScript based stored procedures.

