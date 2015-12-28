---
layout: post
title:  "Quick reference for Oracle stored outlines"
date:   2013-03-12 12:00:28
---

Sometimes when you are faced with Oracle performance issues in production, the quick remedy is to use stored outlines to tune SQL execution plan without the need to alter the code.

A stored outline is a collection of hints associated with a specific SQL statement that allows a standard execution plan to be maintained, regardless of changes in the system environment or associated statistics. Plan stability is based on the preservation of execution plans at a point in time where the performance of a statement is considered acceptable.

Below is a quick reference how-to use Oracle outlines to tune problematic SQL queries.

Imagine we have a single query which is using a BAD plan in the production database. We also have the SQL\_ID, SQL\_HASH\_VALUE, SQL\_CHILD\_NUMBER of this query which is using the BAD plan. On the other side we can either tune it by adding HINTS or have another version of this same query which is running fine in the same production database with different BIND variables.

```
-- ======================
-- Enable stored outlines
-- ======================
ALTER SESSION SET query_rewrite_enabled=TRUE;
 
-- ======================
-- Bad and good queries
-- ======================
select * from rk13.T_SOTEST where owner ='SYS';      -- bad execution plan
select * from rk13.T_SOTEST where owner ='TSMSYS';   -- good execution plan
  
-- ======================
-- Get sql hash for queries
-- ======================
SELECT hash_value, child_number, sql_text FROM v$sql WHERE sql_text LIKE 'select * from rk13.T_SOTEST where owner =%';
   
-- ======================
-- Prepare outlines
-- ======================
BEGIN
DBMS_OUTLN.create_outline(
hash_value    => 2956310531,
child_number  => 0);
END;
/
    
-- ======================
-- Check if outline is used
-- ======================
SELECT name, category, used, sql_text FROM user_outlines;
     
-- ======================
-- Rename outline
-- ======================
ALTER outline SYS_OUTLINE_12101800110706711 rename to T_SOTEST_BAD;
SELECT node, stage, join_pos, hint  FROM user_outline_hints WHERE name = 'T_SOTEST_BAD';
      
-- ======================
-- Create private outlines
-- ======================
create private outline PRIVOL_BAD from T_SOTEST_BAD;
create private outline PRIVOL_GOOD from T_SOTEST_GOOD;
       
-- ======================
-- Swap outline hints
-- ======================
UPDATE system.ol$hints SET ol_name = decode(ol_name, 'PRIVOL_BAD','PRIVOL_GOOD','PRIVOL_GOOD','PRIVOL_BAD') WHERE ol_name in ('PRIVOL_GOOD','PRIVOL_BAD');
        
-- ======================
-- Refresh and replace outline
-- ======================
execute dbms_outln_edit.refresh_private_outline('PRIVOL_BAD');
execute dbms_outln_edit.refresh_private_outline('PRIVOL_GOOD');
create or replace outline T_SOTEST_BAD from private PRIVOL_BAD;
```
https://gist.github.com/rk13/5130698#file-oracle-tune-outline

http://oracle-knowledgemine.blogspot.com/2009/06/stored-outline-in-oracle-10g.html

http://www.oracle-base.com/articles/misc/outlines.php

http://it.toolbox.com/blogs/living-happy-oracle/oracle-need-a-hint-use-sql-profiles-instead-of-stored-outlines-29638

http://www.dba-oracle.com/t_swapping_sql_profiles.htm

http://www.dba-oracle.com/t_cbo_stored_outlines.htmi
