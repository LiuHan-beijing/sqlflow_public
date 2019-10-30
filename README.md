## SQLFlow

SQLFlow is a tool that automates data lineage discovery by analyzing the SQL script.
It generates a nice clean diagram to show the dataflow among the table/view and columns
in the data warehouse.

Support more than 20 major databases including bigquery, couchbase, dax, db2, 
greenplum, hana, hive, impala, informix, mdx, mysql, netezza, openedge, oracle, postgresql, 
redshift, snowflake, sqlserver, sybase, teradata, vertica,

Just paste the SQL script and click a button, you will get the data lineage diagram instantly,
highlight the dataflow in the diagram with a simple mouse click.

You can also call the RESTful API provided by this tool in your own program and 
get the data lineage and diagram model information in a JSON snippet to make further usage.

### SQLFlow Live 
[SQLFlow Live](https://www.gudusoft.com/sqlflow)


![SQLFlow architecture](sqlflow_architecture.png)

### SQLFlow components 
![SQLFlow components](sqlflow_components.png)

#### SQLFlow frontend
1. Send the SQL script received from the browser in JSON to the backend.

2. After receiving the result which includes the data lineage and diagram model 
generated by the backend, visualize the diagram model in the browser.

3. Highlight the dataflow in the diagram when the user clicks on a specific column.

#### SQLFlow backend
1. `SQLFlow`: receiving the SQL script from the frontend and parse the SQL script into parse tree nodes
by utilizing [the GSP library](http://www.sqlparser.com), then calculate the dlineage by analyzing AST.

2. `FlowLayout`:  Calculating the layout of database objects(table/column) in the dlineage and 
 generate the diagram model with all necessary position data, including nodes and edges.
 `FlowLayout` depends on `doLayout library` to layout the database objects.

3. Return a JSON snippet including the data lineage and diagram model to the frontend.


### SQLFlow RESTful API

SQLFlow also provides RESTful API, so your program can communicate with the SQLFlow backend directly.
Sending the SQL to SQLFlow backend and receive a JSON snippet including the data lineage and diagram model
for further processing in your own program.


## Type of the column relationships in JSON snippet

* **fdd**,  the value of target column is come from source column, such as: t = s + 2
	
	You may check `effectType` to see how the target column is changed.
	
	- `effectType = select`, the source data is from select.
	- `effectType = insert`, the source data is from insert.
	- `effectType = update`, the source data is from update.
	- `effectType = merge_update`, the source data is merge update.
	- `effectType = merege_insert`, the source data is from merege insert.
	- `effectType = create_view`, the source data is from create view.
		
* **fddi**, the value of the target column is not derived from the source column directly, but it is affected by the source column.
		However, it's difficult to determine this kind of relation, take this syntax for example: t=fx(s).
		so the relationship of the source and target column is `fdd` in a function call unless we know clearly that `t` will not 
		include value from `s`.
		
	In this sample SQL, the relationship between `teur` and `kamut` is fddi.

	```sql
	select
		case when a.kamut=1 and b.teur IS null
				 then 'no locks'
			 when a.kamut=1
				then b.teur
		else 'locks'
		end teur
	from tbl a left join TT b on (a.key=b.key)
	```
		
	
		
* **fdr**, the value of target column affected by the row number of the source table. It always happens when the aggregate function is used.
	the source column may be appeared in the where，group by clause. This kind of relationship may also appear between the target column and the source table.
	
	
* **frd**, the row number of target column is affected by the value of source column. The source column usually appears in the where clause.
	
		
The meaning of the letter in `fdd`, `fddi`, `fdr`, `frd`, f: dataflow, d: data value, r: record set.

The first letter is always f，the second letter represents the target column，the third letter represents the source column, the fourth is reserved.


### fdd
The value of the target column is derived from the source column directly.
```sql
create view vEmp(eName) as
SELECT a.empName "eName"
FROM scott.emp a
Where sal > 1000
```

`vEmp.eName` <- fdd <- `"eName"` <- fdd <- `scott.emp.empName`

### fdr
The value of the target column is influenced by a source table itself, for example by the number of records. 
This is caused by the use of aggregate function in the query.
```sql
create view vSal as
SELECT a.deptno "Department", 
    a.num_emp/b.total_count "Employees", 
    a.sal_sum/b.total_sal "Salary"
FROM
    (SELECT deptno, COUNT() num_emp, SUM(SAL) sal_sum
        FROM scott.emp
       Where city = 'NYC'
       GROUP BY deptno) a,
    (SELECT COUNT() total_count, SUM(sal) total_sal
        FROM scott.emp
       Where city = 'NYC') b
```

`vSal.Salary` depends on the record number of table: `scott.emp`. 
This is due to the use of aggregate function `COUNT()`, `SUM()` in the query, 
`vSal.Salary` also depends on the `scott.emp.deptno` column which is used in the 
group by clause.

The `city` column in the where clause also determines the value of `vSal.Salary`.

The chain of the dataflow is:

`vSal.Salary` <- fdd <- `query_resultset.Salary`

`query_resultset.Salary` <- fdd <- `sal_sum`

`query_resultset.Salary` <- fdd <- `total_sal`

`sal_sum` <- fdd <- `scott.emp.SAL`

`sal_sum` <- fdr <- `scott.emp`

`sal_sum` <- fdr <- `scott.emp.city`

`sal_sum` <- fdr <- `scott.emp.deptno`

`total_sal` <- fdd <- `scott.emp.sal`

`total_sal` <- fdr <- `scott.emp`

`total_sal` <- fdr <- `scott.emp.city`


### frd
Some of the columns in source tables such as WHERE clause do not influence the value of target columns 
but are crucial for the selected row set, so they are also saved for impact analyses, 
with relationship to the target columns.
```sql
create view vEmp(eName) as
SELECT a.empName "eName"
FROM scott.emp a
Where sal > 1000
```
The value of `vEmp.eName` doesn’t depends on `scott.emp.sal`, 
but the number of records in the `vEmp` depends on the `scott.emp.sal`, 
so this tool record this kind of relationship as well

`vEmp.eName` <- fdd <- `query_resultset."eName"` <- fdd <- `scott.emp.empName`

`vEmp.eName` <- fdd <- `query_resultset."eName"` <- frd <- `scott.emp.sal`

### fddi
The value of the target column depends on the value of the source column, but not come from the source column.
```sql
select
	case when a.kamut=1 and b.teur IS null
			 then 'no locks'
		 when a.kamut=1
			then b.teur
	else 'locks'
	end teur
from tbl a left join TT b on (a.key=b.key)
```

The value of the select result: `teur` depends on the source column `tbl.kamut` 
in the case expression, although it values is not derived from `tbl.kamut` directly.

`query_result.teur` <- fddi <- `tbl.kamut`

`query_result.teur` <- frd <- `tbl.key`

`query_result.teur` <- frd <- `TT.key`

