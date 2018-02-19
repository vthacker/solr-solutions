# Capturing Solr Metrics To a Log File

Solr 6.4+ has the ability to capture a lot of metrics. The [Metrics Reporting](http://lucene.apache.org/solr/guide/metrics-reporting.html) page gives a comprehensive introduction  


# Goal
Demonstrate Solr's SQL capabilities 


# MySQL Employees Data
We use the MySQL employees data example and load it in Solr

```properties
Download SQL data repository as a zip file : https://github.com/datacharmer/test_db
From data repository run: mysql < employees.sql
Docs about the data is present here : https://dev.mysql.com/doc/employee/en/
The database schema is available on : https://dev.mysql.com/doc/employee/en/sakila-structure.html
```

# Start Solr

```properties
./bin/solr start -c -m 4g
```



# Create departments collection and load data

```bash
./bin/solr create -c departments

curl --data-urlencode 'expr=
commit(
    departments,
    update(departments,
        batchSize=500,
        jdbc(
            connection="jdbc:mysql://localhost:3306/employees?user=root",
            sql="select dept_no as dept_no_s, dept_name as dept_name_s, dept_name as dept_name_txt from departments",
            sort="dept_no asc",
            driver="com.mysql.jdbc.Driver",
            get_column_name="false"
        )
    )
)' http://localhost:8983/solr/departments/stream
```

# Create department manager collection and load data

```bash
./bin/solr create -c dept_manager

curl --data-urlencode 'expr=
commit(
    dept_manager,
    update(dept_manager,
        batchSize=500,
        jdbc(
            connection="jdbc:mysql://localhost:3306/employees?user=root",
            sql="select emp_no as emp_no_s, dept_no as dept_no_s, from_date as from_date_dt, to_date as to_date_dt from dept_manager",
            sort="dept_no asc",
            driver="com.mysql.jdbc.Driver",
            get_column_name="false"
        )
    )
)' http://localhost:8983/solr/dept_manager/stream
```

# Create employees collection and load data

```bash
./bin/solr create -c employees

curl --data-urlencode 'expr=
commit(
    employees,
    update(employees,
        batchSize=5000,
        jdbc(
            connection="jdbc:mysql://localhost:3306/employees?user=root",
            sql="select emp_no as emp_no_s, birth_date as birth_date_dt, first_name as first_name_s, last_name as last_name_s, gender as gender_s, hire_date as hire_date_dt from employees",
            sort="emp_no asc",
            driver="com.mysql.jdbc.Driver",
            get_column_name="false"
        )
    )
)' http://localhost:8983/solr/employees/stream
```

# Create titles collection and load data

```bash
./bin/solr create -c titles

curl --data-urlencode 'expr=
commit(
    titles,
    update(titles,
        batchSize=2000,
        jdbc(
            connection="jdbc:mysql://localhost:3306/employees?user=root",
            sql="select emp_no as emp_no_s, title as title_s, title as title_txt, from_date as from_date_dt, to_date as to_date_dt from titles",
            sort="emp_no asc",
            driver="com.mysql.jdbc.Driver",
            get_column_name="false"
        )
    )
)' http://localhost:8983/solr/titles/stream
```

# Create salaries collection and load data

```bash
./bin/solr create -c salaries

curl --data-urlencode 'expr=
commit(
    salaries,
    update(salaries,
        batchSize=2000,
        jdbc(
            connection="jdbc:mysql://localhost:3306/employees?user=root",
            sql="select emp_no as emp_no_s, salary as salary_i, from_date as from_date_dt, to_date as to_date_dt from salaries",
            sort="emp_no asc",
            driver="com.mysql.jdbc.Driver",
            get_column_name="false"
        )
    )
)' http://localhost:8983/solr/salaries/stream
```

# Create department employees collection and load data

```bash
./bin/solr create -c dept_emp

curl --data-urlencode 'expr=
commit(
    dept_emp,
    update(dept_emp,
        batchSize=2000,
        jdbc(
            connection="jdbc:mysql://localhost:3306/employees?user=root",
            sql="select emp_no as emp_no_s, dept_no as dept_no_s, from_date as from_date_dt, to_date as to_date_dt from dept_emp",
            sort="emp_no asc",
            driver="com.mysql.jdbc.Driver",
            get_column_name="false"
        )
    )
)' http://localhost:8983/solr/dept_emp/stream
```

# Apache Zeppelin Setup


Create a query collection in Solr which we will use as the coordinator for the SQL Queries

```bash
./bin/solr create -c employee_query_collection
```

Setup Apache Zeppelin JDBC Source Using : http://lucene.apache.org/solr/guide/solr-jdbc-apache-zeppelin.html

The interpreter settings used are

```bash
Name : Solr
Interpreter : jdbc
default.url : jdbc:solr://localhost:9983?collection=employee_query_collection
default.driver : org.apache.solr.client.solrj.io.sql.DriverImpl
default.user : solr
dependency : org.apache.solr:solr-solrj:7.2.0

```

Create a Zeppelin Notebook to execute SQL Queries

# SQL Queries


Find 10 top earning employee names

```sql
select s.emp_no_s,s.salary_i,e.first_name_s,e.last_name_s,e.gender_s from 
    (select emp_no_s,salary_i from salaries
    order by salary_i desc
    limit 10) s
INNER JOIN employees e
ON s.emp_no_s = e.emp_no_s
limit 10;


TODO: Why isn't this working 

select s.emp_no_s,s.salary_max,e.first_name_s,e.last_name_s,e.gender_s from 
    (select emp_no_s,max(salary_i) as salary_max from salaries 
    group by emp_no_s
    order by salary_max desc
    limit 10) s
INNER JOIN employees e
ON s.emp_no_s = e.emp_no_s
limit 10;
```

# Further research

1. error handling on empty collection throws a confusion exception  :
   Error while executing SQL "select id from gettingstarted": From line 1, column 8 to line 1, column 9: Column 'id' not found in any table
   
2. how to create views to cache common joins to execute query on? Possible with a combination of streaming+sql I think

3. How do we deal with multi-valued fields? PS: I haven't tested it yet

4. Couldn't get back a _txt field . https://issues.apache.org/jira/browse/SOLR-8362 might fix it?

5. Handling of {{SELECT * }} is not supported. How many tools rely on this? 