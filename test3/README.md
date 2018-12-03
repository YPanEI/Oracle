# 实验三：创建分区表
### 实验目的：掌握分区表的创建方法，掌握各种分区方式的使用场景。
### 实验目的： 
- 本实验使用3个表空间：USERS,USERS02,USERS03。在表空间中创建两张表：订单表(orders)与订单详表(order_details)。
- 使用你自己的账号创建本实验的表，表创建在上述3个分区，自定义分区策略。
- 你需要使用system用户给你自己的账号分配上述分区的使用权限。你需要使用system用户给你的用户分配可以查询执行计划的权限。
- 表创建成功后，插入数据，数据能并平均分布到各个分区。每个表的数据都应该大于1万行，对表进行联合查询。
- 写出插入数据的语句和查询数据的语句，并分析语句的执行计划。
- 进行分区与不分区的对比实验。
### 实验相关：
- 实验人：杨洋
- 用户名：ypan_user 角色ypan_view
- 学号：201610414424
- 实验日期：2018.10.8
#### 实验步骤：
##### 步骤一：创建orders（订单表）
```sql
CREATE TABLE orders 
 (
  order_id NUMBER(10, 0) NOT NULL  primary key
  , customer_name VARCHAR2(40 BYTE) NOT NULL 
  , customer_tel VARCHAR2(40 BYTE) NOT NULL 
  , order_date DATE NOT NULL 
  , employee_id NUMBER(6, 0) NOT NULL 
  , discount NUMBER(8, 2) DEFAULT 0 
  , trade_receivable NUMBER(8, 2) DEFAULT 0 
 ) 
 TABLESPACE USERS
 PCTFREE 10 INITRANS 1
 STORAGE (   BUFFER_POOL DEFAULT )
 NOCOMPRESS NOPARALLEL
 PARTITION BY RANGE (order_date)
 (
  PARTITION PARTITION_BEFORE_2017 VALUES LESS THAN (
  TO_DATE(' 2017-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN'))
  NOLOGGING
  TABLESPACE USERS
  PCTFREE 10
  INITRANS 1
  STORAGE
 (
  INITIAL 8388608
  NEXT 1048576
  MINEXTENTS 1
  MAXEXTENTS UNLIMITED
  BUFFER_POOL DEFAULT
 )
 NOCOMPRESS NO INMEMORY
 , PARTITION PARTITION_BEFORE_2018 VALUES LESS THAN (
 TO_DATE(' 2018-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS',
 'NLS_CALENDAR=GREGORIAN'))
 NOLOGGING
 TABLESPACE USERS02
  PCTFREE 10
  INITRANS 1
  STORAGE
 (
  INITIAL 8388608
  NEXT 1048576
  MINEXTENTS 1
  MAXEXTENTS UNLIMITED
  BUFFER_POOL DEFAULT
 )
 NOCOMPRESS NO INMEMORY
 , PARTITION PARTITION_BEFORE_2019 VALUES LESS THAN (
 TO_DATE(' 2019-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS',
 'NLS_CALENDAR=GREGORIAN'))
 NOLOGGING
 TABLESPACE USERS03
  PCTFREE 10
  INITRANS 1
  STORAGE
 (
  INITIAL 8388608
  NEXT 1048576
  MINEXTENTS 1
  MAXEXTENTS UNLIMITED
  BUFFER_POOL DEFAULT
 )
 NOCOMPRESS NO INMEMORY
 );
 ```
 ##### 步骤二：创建order_details（订单详表）
 
 ```sql
 CREATE TABLE order_details
(
id NUMBER(10, 0) NOT NULL
, order_id NUMBER(10, 0) NOT NULL
, product_id VARCHAR2(40 BYTE) NOT NULL
, product_num NUMBER(8, 2) NOT NULL
, product_price NUMBER(8, 2) NOT NULL
, CONSTRAINT order_details_fk1 FOREIGN KEY  (order_id)
REFERENCES orders  (order_id)
ENABLE
)
TABLESPACE USERS
PCTFREE 10 INITRANS 1
STORAGE (   BUFFER_POOL DEFAULT )
NOCOMPRESS NOPARALLEL
PARTITION BY REFERENCE (order_details_fk1)
(
PARTITION PARTITION_BEFORE_2016
NOLOGGING
TABLESPACE USERS
PCTFREE 10
 INITRANS 1
 STORAGE
(
 INITIAL 8388608
 NEXT 1048576
 MINEXTENTS 1
 MAXEXTENTS UNLIMITED
 BUFFER_POOL DEFAULT
)
NOCOMPRESS NO INMEMORY,
PARTITION PARTITION_BEFORE_2017
NOLOGGING
TABLESPACE USERS02
PCTFREE 10
 INITRANS 1
 STORAGE
(
 INITIAL 8388608
 NEXT 1048576
 MINEXTENTS 1
 MAXEXTENTS UNLIMITED
 BUFFER_POOL DEFAULT
)
NOCOMPRESS NO INMEMORY,
PARTITION PARTITION_BEFORE_2018
NOLOGGING
TABLESPACE USERS03
PCTFREE 10
 INITRANS 1
 STORAGE
(
 INITIAL 8388608
 NEXT 1048576
 MINEXTENTS 1
 MAXEXTENTS UNLIMITED
 BUFFER_POOL DEFAULT
)
NOCOMPRESS NO INMEMORY
);
```
 - **表创建成功，如下图所示：**<br><br>
 ![表创建成功](https://github.com/YPanEI/Oracle/blob/master/test3/images/1.png)
 
 ##### 查看数据库使用情况
 $ sqlplus system/123@pdborcl
 ```sql
 SQL>SELECT tablespace_name,FILE_NAME,BYTES/1024/1024 MB,MAXBYTES/1024/1024 MAX_MB,autoextensible FROM dba_data_files  WHERE  tablespace_name='USERS';

SQL>SELECT a.tablespace_name "表空间名",Total/1024/1024 "大小MB",
 free/1024/1024 "剩余MB",( total - free )/1024/1024 "使用MB",
 Round(( total - free )/ total,4)* 100 "使用率%"
 from (SELECT tablespace_name,Sum(bytes)free
        FROM   dba_free_space group  BY tablespace_name)a,
       (SELECT tablespace_name,Sum(bytes)total FROM dba_data_files
        group  BY tablespace_name)b
 where  a.tablespace_name = b.tablespace_name;
 ```
##### 步骤三：插入一万条数据
```sql
create table MIS_SEC_USER(USERID number, USERNAME varchar(32), PWD varchar(32))
declare
v_uid number;
v_usr varchar(32);
v_pwd varchar(32);
v_fg1 number;
v_fg2 number;
begin
v_fg1 := 0;
loop
v_uid := abs(dbms_random.random());
v_usr := dbms_random.string(opt => 'u', len => 10);
v_pwd := dbms_random.string(opt => 'u', len => 10);
select count(1) into v_fg2 from MIS_SEC_USER where USERID = v_uid or USERNAME = v_usr;
if v_fg2 = 0 then
insert into MIS_SEC_USER values(v_uid, v_usr, v_pwd);
v_fg1 := v_fg1 + 1;
end if;
exit when v_fg1 >= 10000;
end loop;
commit;
end;
```
