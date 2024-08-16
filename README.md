# 目的
- SQLの実行計画の見方
- インデックスの作成方法と効果測定

# 前提条件
こちらのカリキュラムは前回の`oracle-study`で使用した環境を前提としています。
```
https://github.com/NakamuraYosuke/oracle-study
```

# 事前準備
## 権限付与
```
sqlplus sys@ORACLE as sysdba
```

```
SQL> @?/sqlplus/admin/plustrce
```

## 共有プールのクリア
```
SQL> ALTER SYSTEM FLUSH SHARED_POOL;
```

# 改善前の実行計画の取得
```
SQL> CONN HR@ORACLE
```

```
SQL> SET AUTOTRACE ON
```

```
SQL> SELECT EMPLOYEE_ID,FIRST_NAME,LAST_NAME,SALARY FROM EMPLOYEES WHERE SALARY = (SELECT MAX(SALARY) FROM EMPLOYEES);
```

```
Execution Plan
----------------------------------------------------------
Plan hash value: 1945967906

---------------------------------------------------------------------------------
| Id  | Operation	    | Name	| Rows	| Bytes | Cost (%CPU)| Time	|
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |		|     1 |    52 |     6   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL  | EMPLOYEES |     1 |    52 |     3   (0)| 00:00:01 |
|   2 |   SORT AGGREGATE    |		|     1 |    13 |	     |		|
|   3 |    TABLE ACCESS FULL| EMPLOYEES |   107 |  1391 |     3   (0)| 00:00:01 |
---------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("SALARY"= (SELECT MAX("SALARY") FROM "EMPLOYEES"
	      "EMPLOYEES"))

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)


Statistics
----------------------------------------------------------
	121  recursive calls
	  0  db block gets
	227  consistent gets
	  0  physical reads
	  0  redo size
	804  bytes sent via SQL*Net to client
	474  bytes received via SQL*Net from client
	  2  SQL*Net roundtrips to/from client
	  6  sorts (memory)
	  0  sorts (disk)
	  1  rows processed
```

## 共有プールのクリア
```
SQL> SET AUTOTRACE OFF
SQL> CONN SYS AS SYSDBA
SQL> ALTER SYSTEM FLUSH SHARED_POOL;
```

# インデックスの作成
```
SQL> CONN HR@ORACLE
```

```
CREATE INDEX IDX_EMPLOYEES_SALARY ON EMPLOYEES (SALARY);
```

# 改善後の実行計画の取得
```
SQL> SELECT EMPLOYEE_ID,FIRST_NAME,LAST_NAME,SALARY FROM EMPLOYEES WHERE SALARY = (SELECT MAX(SALARY) FROM EMPLOYEES);
```

```
Execution Plan
----------------------------------------------------------
Plan hash value: 158051583

------------------------------------------------------------------------------------------------------------
| Id  | Operation			    | Name		   | Rows  | Bytes | Cost (%CPU)| Time	   |
------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT		    |			   |	 1 |	52 |	 3   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| EMPLOYEES 	   |	 1 |	52 |	 2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN		    | IDX_EMPLOYEES_SALARY |	 1 |	   |	 1   (0)| 00:00:01 |
|   3 |    SORT AGGREGATE		    |			   |	 1 |	13 |		|	   |
|   4 |     INDEX FULL SCAN (MIN/MAX)	    | IDX_EMPLOYEES_SALARY |	 1 |	13 |	 1   (0)| 00:00:01 |
------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("SALARY"= (SELECT MAX("SALARY") FROM "EMPLOYEES" "EMPLOYEES"))

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)


Statistics
----------------------------------------------------------
	 24  recursive calls
	  0  db block gets
	 35  consistent gets
	  0  physical reads
	  0  redo size
	804  bytes sent via SQL*Net to client
	474  bytes received via SQL*Net from client
	  2  SQL*Net roundtrips to/from client
	  0  sorts (memory)
	  0  sorts (disk)
	  1  rows processed
```
