
## report module ##

- annotation
	- Dimension.java
	- Metric.java
	- ReportEntity.java		dbId，name
	- Time.java
	- UserId.java
- constant
	- ColumnType.java	COL, DIM, MET, USER, TIME;
	- TimeGranularity.java
	
			WARN：语义不清析，将sum与hour/day/week/等放在一起 hour(0),day(1),week(2),month(3), year(4), sum(5);	

- dbms
	- mysql
		- QueryExecutorImpl.java

			INFO：Work getResult, getSql

		- SimpleDateSource.java
		- SqlBuilder.java	生成SQL的工具，对Query进行转化， dim, metric, condiction, order, 
	- QueryExecutor.java
			
			INFO: 查询报表生成的Frame对象，查询总数，查询接口

- serial
	- CsvSerial.java
	- Serial.java
- sql
	- criterion
		- Criterion.java		条件
		- Limit.java			分页使用
		- Order.java
		- Resrictions.java   where条件
	- fields
		- Dim.java		维度（广告维度）
		- Field.java		域（类，原名，表中的域名，列的意义）
		- Metric.java	继承域，Metric类型（值）
		- Time.java		时间，特殊处理
		- UserId.java	用户ID，
	- Column.java
	- ColumnComputer.java
	- ColumnMeta.java  列属性（包含ColumnType)
	- Entity.java   Table对像实体，包含了 Dim，Field, Metric，Time，UserId，dbid, name
	- Frame.java		返回的结果，包含Table，MetaMap
	- Pair.java
	- **Query.java** 查询条件的封装，提供了一个build方法，和add limit，包含了 metric list, order list, dim list, start, end, userid, entity, criterions, 粒度
	
- utils
	- DateBuilder.java
	- DateFormatProvider.java		
	- FieldProvider.java
	- FrameFiller.java
	- NumberUtils.java
	- TwoColumnDivideComputer.java

## SQL资料 ##

ResultSetMetaData 只是获得表的信息