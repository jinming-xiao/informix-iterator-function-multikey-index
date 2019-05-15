<h1>informix multi-key (iterator) function index</h1>

multi-key (iterator) function index in 14.10 is a framework to build index on iterator udr function, users can use index scan to query collections data types’ elements and JSON/BSON array.

<h3>example to show how this feature works</h3>
<b>1) create table with a set collumn</b>
<pre>
CREATE TABLE test(city char(16), zipcode set(varchar(255) not null));

insert into test values ("Overland Park", 
	set{"66213", "66214", "66202"});
insert into test values ("Lenexa", set{ "66214", "66229"});
insert into test values ("Olathe", set{ "66051", "66061"});
insert into test values ("Shawnee", set{ "66201", "66202", "66203"});
</pre>
<b>2) create an iterator functions to access a set</b>
<pre>
CREATE FUNCTION informix.any1(s set(varchar(255) not null))
        RETURNS char(5) with (not variant);
  DEFINE zip char(5);
  FOREACH cursor1 FOR
    SELECT * into zip FROM TABLE(s)
    return zip with resume;
  END FOREACH ;
END FUNCTION;
</pre>
<b>3) create function index on collection data type</b>

<pre>
CREATE INDEX zipx on test(any1(zipcode));
</pre>

<b>4) query collection type elements</b> 
<pre>
select * from test where any1(zipcode) = '66214’;
	city     Overland Park
	zipcode  SET{'66213','66214','66202’}

	city     Lenexa
	zipcode  SET{'66214','66229'}

Estimated Cost: 1
Estimated # of Rows Returned: 1
  1) informix.test: INDEX PATH
    (1) Index Keys: informix.any1(zipcode)   (Serial, fragments: ALL)
        Lower Index Filter: informix.any1(informix.test.zipcode )= '66214'

 Table map :
  ----------------------------
  Internal name     Table name
  ----------------------------
  t1                test
  type     table  rows_prod  est_rows  rows_scan  time       est_cost
  -------------------------------------------------------------------
  scan     t1     2          1         2          00:00.00   1

without this feature, sequential scan is used
select * from test where '66214' in zipcode
Estimated Cost: 2
Estimated # of Rows Returned: 1
  1) informix.test: SEQUENTIAL SCAN
        Filters: '66214' IN (informix.test.zipcode )

 Table map :
  ----------------------------
  Internal name     Table name
  ----------------------------
  t1                test
  type     table  rows_prod  est_rows  rows_scan  time       est_cost
  -------------------------------------------------------------------
  scan     t1     2          1         4          00:00.00   2       

</pre>

<h3>IDS 14.10 built-in Iterator UDRs</h3>
Informix 14.10 has a couple of built-in iterator UDRs for better support multi-key function index, they're included in $INFORMIXDIR/etc/boot14.10UC1.sql
<pre>
any(multiset/set/list(int not null))
any(multiset/set/list(varchar(255) not null))
any(multiset/set/list(date not null))
any(multiset/set/list(datatime year to second not null))
any(multiset/set/list(float not null))
any(multiset/set/list(bigint not null)) 
create dba function informix.any(multiset(int not null))
        returns int with (iterator)
        external name '(collany_int)' language C not variant;

use the built-in udrs 

drop index if exists zipx;
create index zipx on test(any(zipcode));
select * from test where any(zipcode) = '66214';

</pre>

<h3>index on BSON array elements</h3>
<b>built-in BSON array iterator C-UDRs</b>
<pre>
bson_mvalue_varchar(informix.bson, informix.lvarchar)
bson_mvalue_int(informix.bson, informix.lvarchar)
bson_mvalue_bigint(informix.bson, informix.lvarchar)
bson_mvalue_double(informix.bson, informix.lvarchar)
bson_mvalue_date(informix.bson, informix.lvarchar)
</pre>

<b>example to create index on BSON array </b>
<pre>
create table tab (id int, parking bson);
create index idx on tab(bson_mvalue_varchar(parking, ”cars"));
insert into tab values (1, ‘{“cars":[”ford",”audi",”toyota"]}'::JSON::BSON);
insert into tab values (2, ‘{“cars":[”ford",”BMW"]}'::JSON::BSON);
insert into tab values (3, ‘{“cars":[”BMW",”Toyota", ”Dodge"]}'::JSON::BSON);

select id, parking::json from tab 
	where bson_mvalue_varchar(parking, ”cars") = ‘Ford’;

id            1
(expression)  {"cars":["Ford","Audi","Toyota"]} 
id            2
(expression)  {"cars":["Ford","BMW"]} 
2 row(s) retrieved.

</pre>

