---
id: extend-jdbc  
title: JDBC Protocol Custom Monitoring  
sidebar_label: JDBC Protocol Custom Monitoring    
---
> From [Custom Monitoring](extend-point), you are familiar with how to customize types, Metrics, protocols, etc. Here we will introduce in detail how to use JDBC(support mysql,mariadb,postgresql,sqlserver at present) to customize Metric monitoring. 
> JDBC protocol custom monitoring allows us to easily monitor Metrics we want by writing SQL query statement.   

### JDBC protocol collection process    
【**System directly connected to MYSQL**】->【**Run SQL query statement**】->【**parse reponse data: oneRow, multiRow, columns**】->【**Metric data extraction**】   

It can be seen from the process that we define a monitoring type of JDBC protocol. We need to configure SSH request parameters, configure which Metrics to obtain, and configure query SQL statements.

### Data parsing method   
We can obtain the corresponding Metric data through the data fields queried by SQL and the Metric mapping we need. At present, there are three mapping parsing methods：oneRow, multiRow, columns.

#### **oneRow**  
> Query a row of data, return the column name of the result set through query and map them to the queried field.  

eg：   
queried Metric fields：one two three four
query SQL：select one, two, three, four from book limit 1;
Here the Metric field and the response data can be mapped into a row of collected data one by one.     

#### **multiRow**
> Query multiple rows of data, return the column names of the result set and map them to the queried fields.  

eg：   
queried Metric fields：one two three four
query SQL：select one, two, three, four from book;
Here the Metric field and the response data can be mapped into multiple rows of collected data one by one. 

#### **columns**
> Collect a row of Metric data. By matching the two columns of queried data (key value), key and the queried field, value is the value of the query field. 

eg：   
queried fields：one two three four   
query SQL：select key, value from book;   
SQL response data：   

| key      | value |
| ----------- | ----------- |
| one    | 243 |
| two    | 435 |
| three  | 332 |
| four   | 643 |

Here by mapping the Metric field with the key of the response data, we can  obtain the corresponding value as collection and monitoring data.

### Custom Steps  

In order to configure a custom monitoring type, you need to add and configure two YML file.
1. Monitoring configuration definition file named after monitoring type - eg：example_sql.yml in the installation directory /hertzbeat/define/app/
2. Monitoring parameter definition file named after monitoring type - eg ：example_sql.yml in the installation directory /hertzbeat/define/param/
3. Restart hertzbeat system, we successfully fit a new custom monitoring type.

------- 
Configuration usages of the two files are detailed below. Please pay attention to usage annotation.   

### Monitoring configuration definition file   

> Monitoring configuration definition file is used to define *the name of monitoring type(international), request parameter mapping, index information, collection protocol configuration information*, etc.  

eg：Define a custom monitoring type named example_sql which use the JDBC protocol to collect data.    
The file name: example_sql.yml in /define/app/example_sql.yml   

```yaml
# The monitoring type category：service-application service monitoring db-database monitoring custom-custom monitoring os-operating system monitoring
category: db
# Monitoring application type(consistent with the file name) eg: linux windows tomcat mysql aws...
app: example_sql
name:
  zh-CN: 模拟MYSQL应用类型
  en-US: MYSQL EXAMPLE APP
# parameter mapping map. These are input parameter variables which can be written to the configuration in form of ^_^host^_^. The system automatically replace variable's value.
# type means parameter type: 0-number number, 1-string cleartext string, 2-secret encrypted string
# required parameters - host
configmap:
  - key: host
    type: 1
  - key: port
    type: 0
  - key: username
    type: 1
  - key: password
    type: 2
  - key: database
    type: 1
  - key: url
    type: 1
# Metric group list
metrics:
  - name: basic
    # The smaller Metric group scheduling priority(0-127), the higher the priority. After completion of the high priority Metric group collection,the low priority Metric group will then be scheduled. Metric groups with the same priority  will be scheduled in parallel.
    # Metric group with a priority of 0 is an availability group which will be scheduled first. If the collection succeeds, the  scheduling will continue otherwise interrupt scheduling.
    priority: 0
    # Specific monitoring Metrics in the Metric group
    fields:
      # Metric information include field: name   type: field type(0-number: number, 1-string: string)   instance: primary key of instance or not   unit: Metric unit
      - field: version
        type: 1
        instance: true
      - field: port
        type: 1
      - field: datadir
        type: 1
      - field: max_connections
        type: 0
    # (optional)Monitoring Metric alias mapping to the Metric name above. The field used to collect interface data is not the final Metric name directly. This alias is required for mapping conversion.
    aliasFields:
      - version
      - version_compile_os
      - version_compile_machine
      - port
      - datadir
      - max_connections
    # (optional)The Metric calculation expression works with the above alias to calculate the final required Metric value.
    # eg: cores=core1+core2, usage=usage, waitTime=allTime-runningTime
    calculates:
      - port=port
      - datadir=datadir
      - max_connections=max_connections
      - version=version+"_"+version_compile_os+"_"+version_compile_machine
    protocol: jdbc
    jdbc:
      # host: ipv4 ipv6 domain name
      host: ^_^host^_^
      # port
      port: ^_^port^_^
      platform: mysql
      username: ^_^username^_^
      password: ^_^password^_^
      database: ^_^database^_^
      # SQL query method：oneRow, multiRow, columns
      queryType: columns
      # sql
      sql: show global variables where Variable_name like 'version%' or Variable_name = 'max_connections' or Variable_name = 'datadir' or Variable_name = 'port';
      url: ^_^url^_^

  - name: status
    priority: 1
    fields:
      # Metric information include field: name   type: field type(0-number: number, 1-string: string)   instance: primary key of instance or not   unit: Metric unit
      - field: threads_created
        type: 0
      - field: threads_connected
        type: 0
      - field: threads_cached
        type: 0
      - field: threads_running
        type: 0
    # (optional)Monitoring Metric alias mapping to the Metric name above. The field used to collect interface data is not the final Metric name directly. This alias is required for mapping conversion.
    aliasFields:
      - threads_created
      - threads_connected
      - threads_cached
      - threads_running
    # (optional)The Metric calculation expression works with the above alias to calculate the final required Metric value.
    # eg: cores=core1+core2, usage=usage, waitTime=allTime-runningTime
    calculates:
      - threads_created=threads_created
      - threads_connected=threads_connected
      - threads_cached=threads_cached
      - threads_running=threads_running
    protocol: jdbc
    jdbc:
      # host: ipv4 ipv6 domain name
      host: ^_^host^_^
      # port
      port: ^_^port^_^
      platform: mysql
      username: ^_^username^_^
      password: ^_^password^_^
      database: ^_^database^_^
      # SQL query method: oneRow, multiRow, columns
      queryType: columns
      # sql
      sql: show global status where Variable_name like 'thread%' or Variable_name = 'com_commit' or Variable_name = 'com_rollback' or Variable_name = 'questions' or Variable_name = 'uptime';
      url: ^_^url^_^

  - name: innodb
    priority: 2
    fields:
      # Metric information include field: name   type: field type(0-number: number, 1-string: string)   instance: primary key of instance or not   unit: Metric unit
      - field: innodb_data_reads
        type: 0
        unit: times
      - field: innodb_data_writes
        type: 0
        unit: times
      - field: innodb_data_read
        type: 0
        unit: kb
      - field: innodb_data_written
        type: 0
        unit: kb
    protocol: jdbc
    jdbc:
      # host: ipv4 ipv6 domain name
      host: ^_^host^_^
      # port
      port: ^_^port^_^
      platform: mysql
      username: ^_^username^_^
      password: ^_^password^_^
      database: ^_^database^_^
      # SQL query method：oneRow, multiRow, columns
      queryType: columns
      # sql
      sql: show global status where Variable_name like 'innodb%';
      url: ^_^url^_^
```

### Monitoring parameter definition file

> Monitoring parameter definition file is used to define *required input parameter field structure definition (Front-end page render input parameter box according to structure)*.

eg：Define a custom monitoring type named example_sql which use the JDBC protocol to collect data.
The file name: example_sql.yml in /define/param/example_sql.yml   

```yaml
app: example_sql
param:
  - field: host
    name: 
      zh-CN: 主机Host
      en-US: Host
    type: host
    required: true
  - field: port
    name: 
      zh-CN: 端口
      en-US: Port
    type: number
    range: '[0,65535]'
    required: true
    defaultValue: 80
    placeholder: 'Please enter the port'
  - field: database
    name: 
      zh-CN: 数据库名称
      en-US: Database
    type: text
    required: false
  - field: username
    name: 
      zh-CN: 用户名
      en-US: Username
    type: text
    limit: 20
    required: false
  - field: password
    name: 
      zh-CN: 密码
      en-US: Password
    type: password
    required: false
  - field: url
    name: 
      zh-CN: Url
      en-US: Url
    type: text
    required: false
```
