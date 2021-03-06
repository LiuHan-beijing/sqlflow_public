## SQLFlow components

sqlfow was comprised of two parts: frontend and backend. 

### Logical parts
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


### Physical parts


### Install SQLFlow

sqlfow was comprised of two parts: frontend and backend. The frontend and backend
can be installed on the same server, or they can be installed seperated on two different servers.
