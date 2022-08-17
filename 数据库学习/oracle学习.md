[TOC]
# 一、Oracle语法备忘
## 1. dictinct 

*根据行去重* （**注意null会被视为重复值**）

```sql
select dictinct v1 from table1; --根据v1单行去重
select dictinct v1,v2 from table1; --根据v1,v2多行去重
```
***
## 2. ROWNUM ROWID
ROWID:每一行数据的物理地址信息编码而成的一个伪列，
在索引块中，既存储每个索引的键值，也存储具有该键值的行的ROWID。 
**rownum从1开始**，rownum和rowid都是伪列，但是两者的根本是不同的，rownum是根据sql查询出的结果给每行分配一个逻辑编号，所以你的**sql不同也就会导致最终rownum不同**，但是**rowid是物理结构上的**，在每条记录insert到数据库中时，都会有一个**唯一**的物理记录。

```sql
--高效率分页，查询第10-20条数据
SELECT * FROM
	( SELECT ROWNUM AS rowno, g.* FROM G_USER g WHERE ROWNUM <= 20) temp --先查出前20条数据
WHERE temp.rowno >= 10; --再查第10到20条数据
--排序后分页
SELECT * FROM (
SELECT ROWNUM AS rowno,tt.* 
FROM
	(select g.* FROM G_USER g ORDER BY g.CREATE_TIME) tt 
WHERE
	ROWNUM <= 20) temp where temp.rowno >= 10;
--需要注意，若存在排序则当前查询出的ROWNUM是乱序
SELECT ROWNUM as luanxu, g.* FROM G_USER g ORDER BY g.USER_NAME
```
## 3. case when 语法
```sql
--两种写法等效
select CASE sex  
WHEN '1' THEN '男'  
WHEN '2' THEN '女'  
ELSE '其他' END from dual;

select CASE 
WHEN sex='1' THEN '男' --注意一个条件true后就不会进入后面的条件
WHEN sex='2' THEN '女'
ELSE '其他' END from dual;
```
## 4. exists 用法

EXISTS用于检查子查询是否至少会返回一行数据，该子查询实际上并不返回任何数据，只返回true或false

```sql
select * from A a where a.aa in (select b.aa from B b) --适合B小表，结果上等效于（但执行方式不同）
select * from A a where exists(select 1 from B b where b.aa=a.aa) -- 适合A小表，附：select从效率上来说，1>xxx>*，因为不用查字典表。

```
>**in**是把外表和内表作**hash 连接**，而**exists** 是**对外表作loop 循环**，每次loop 循环再对内表进行查询。
>
>如果查询语句使用了not in 那么内外表都进行全表扫描，没有用到索引；而not extists 的子查询依然能用到表上的索引。所以无论那个表大，用not exists 都比not in 要快。
>
>**IN适合于外表大而内表小的情况；EXISTS适合于外表小而内表大的情况；not exists优于not in**
>
>分析一下exists真的就比in的效率高吗？

```sql
我们先讨论IN和EXISTS。 
select * from t1 where x in ( select y from t2 ) 
事实上可以理解为： 
select *  
  from t1, ( select distinct y from t2 ) t2 
 where t1.x = t2.y; 
——如果你有一定的SQL优化经验，从这句很自然的可以想到t2绝对不能是个大表，因为需要对t2进行全表的“唯一排序”，如果t2很大这个排序的性能是不可忍受的。但是t1可以很大，为什么呢？最通俗的理解就是因为t1.x=t2.y可以走索引。但这并不是一个很好的解释。试想，如果t1.x和t2.y都有索引，我们知道索引是种有序的结构，因此t1和t2之间最佳的方案是走merge join。另外，如果t2.y上有索引，对t2的排序性能也有很大提高。 
select * from t1 where exists ( select null from t2 where y = x ) 
可以理解为： 
for x in ( select * from t1 ) 
loop 
   if ( exists ( select null from t2 where y = x.x ) 
   then  
      OUTPUT THE RECORD! 
   end if 
end loop 
——这个更容易理解，t1永远是个表扫描！因此t1绝对不能是个大表，而t2可以很大，因为y=x.x可以走t2.y的索引。
```


    综合以上对IN/EXISTS的讨论，我们可以得出一个基本通用的结论：IN适合于外表大而内表小的情况；EXISTS适合于外表小而内表大的情况。
>
>参考：https://www.cnblogs.com/weifeng123/p/9530758.html
***
## 5. truncate vs delete

 功能上与 delete from <table_name> 相同，都是清空表，**但TRUNCATE更快**，delete可以回滚，**TRUNCATE不能回滚**

```sql
TRUNCATE TABLE <table_name>
```

>（1）DELETE语句执行删除的过程是每次从表中删除一行，并且同时将该行的删除操作作为事务记录在日志中保存以便进行进行回滚操作。
TRUNCATE TABLE 则一次性地从表中删除所有的数据并不把单独的删除操作记录记入日志保存，删除行是不能恢复的。并且在删除的过程中不会激活与表有关的删除触发器。执行速度快。
>
>（2）表和索引所占空间。当表被TRUNCATE 后，这个表和索引所占用的空间会恢复到初始大小，DELETE操作不会减少表或索引所占用的空间。
drop语句将表所占用的空间全释放掉。
>
>（3）一般而言，drop > truncate > delete
>
>（4）应用范围。TRUNCATE 只能对TABLE；DELETE可以是table和view
***
## 6. in 内常量最多1000个
***
## 7. from a,b
不加任何条件的话，from a,b 为**笛卡尔积**，加了条件关联为**内连接**（inner join）
```sql
select count(*) from A; --533
select count(*) from B; --294
select count(*) from A,B; --156702=294*533
select * from A,B where A.a = B.a <=> select * from A inner join B on A.a = B.a --加了条件为内连接
```
***
## 8. between and 语法
```sql
--查找time在2001-1-1 00:00:00到2012-12-12 00:00:00的数据
select * from A a where a.time between date '2001-01-01' and date '2012-12-12';
--查找time不在2001-1-1 00:00:00到2012-12-12 00:00:00的数据
select * from A a where not a.time between date '2001-01-01' and date '2012-12-12';
```
***
## 9. insert select 用法
**只需保证数据个数及类型一致**

```sql
--示例
insert into t_user_role_map(user_id,role_id) select t.id,'aaaa' as role_id from t_user t where length(nvl(t.user_name,''))=20;
```
***
## 10. 表结构及数据快速复制

```sql
--将表AM_PRO_MAG_PERMISSION的结构及数据创建并复制到FANJC_AM_PRO_MAG_PERMISSION中
 CREATE TABLE FANJC_AM_PRO_MAG_PERMISSION AS (SELECT am.* FROM AM_PRO_MAG_PERMISSION am )
```

------

## 11.  union、union all、minus用法

**union**：取并集并去除重复元素

**union all**：取并集不去除重复元素

**minus**：取差集

```sql
select a from table_a union select b from table_b; --获取a,b的并集并去除重复元素
select a from table_a minus select b from tbale_b; --取差集（a-b)
```

## 12. having过滤

having可过滤group by后的数据，也可接聚合函数，判断每组的聚合函数结果是否满足条件。

# 二、Oracle系统级语法

## 1. Oracle热备回滚某个时间点的数据
```sql
-- 查询2017-11-06 13:00:00 时间点，表xxxx_table的中xxxx_id='test'的数据，可用于误操作后回滚数据(利用oracle热备)
select * from <xxxx_table> as of timestamp to_timestamp('2017-11-06 13:00:00', 'yyyy-mm-dd hh24:mi:ss') 
where <xxxx_id> = <value>;
-- 具体回滚语句
delete from <xxxx_table>;
commit;
insert into <xxxx_table> select * from <xxxx_table> as of timestamp to_timestamp('2018-06-08 11:06:00', 'yyyy-MM-dd HH24:mi:ss');
```
***
## 2. index用法
```sql
CREATE INDEX <索引名> ON <表名>(<字段名>);
drop index <索引名>;
CREATE INDEX s_user_username_idx ON s_user(user_name);
select * from user_indexes where table_name = <表名>; --查看某表索引
```
***
## 3. for update用法
```sql
-- for update 会锁住select的row数据，不会锁住整个table，如果for update的事务未提交或撤回，则对锁住的数据执行更新操作会等待row锁释放，
SELECT * from T_BACKAPPLICATION a  where a.STUDENTID ='201606270163' for update;
```
***
## 4. Oracle查看锁表与解锁
事务长时间未提交，超时会导致表锁，所以一个事务内代码不要太复杂，可能导致超时

```
v$locked_object中的LOCKED_MODE字段表示锁的模式，oracle中锁的模式有如下几种:
0：none
1：null 空
2：Row-S 行共享(RS)：共享表锁，sub share
3：Row-X 行独占(RX)：用于行的修改，sub exclusive
4：Share 共享锁(S)：阻止其他DML操作，share
5：S/Row-X 共享行独占(SRX)：阻止其他事务操作，share/sub exclusive
6：exclusive 独占(X)：独立访问使用，exclusive

数字越大锁级别越高, 影响的操作越多。

1级锁有：Select，有时会在v$locked_object出现。
2级锁有：Select for update,Lock For Update,Lock Row Share
select for update当对话使用for update子串打开一个游标时，所有返回集中的数据行都将处于行级(Row-X)独占式锁定，其他对象只能查询这些数据行，不能进行update、delete或select for update操作。
3级锁有：Insert, Update, Delete, Lock Row Exclusive
没有commit之前插入同样的一条记录会没有反应, 因为后一个3的锁会一直等待上一个3的锁, 我们必须释放掉上一个才能继续工作。
4级锁有：Create Index, Lock Share
locked_mode为2,3,4不影响DML(insert,delete,update,select)操作, 但DDL(alter,drop等)操作会提示ora-00054错误。
00054, 00000, “resource busy and acquire with NOWAIT specified”
// *Cause: Resource interested is busy.
// *Action: Retry if necessary.
5级锁有：Lock Share Row Exclusive
具体来讲有主外键约束时update / delete … ; 可能会产生4,5的锁。
6级锁有：Alter table, Drop table, Drop Index, Truncate table, Lock Exclusive
```
>参考：https://blog.csdn.net/u013991521/article/details/53535818
```sql
-- oracle查看被锁表信息
select t2.username,     -- oracle用户名
       t2.sid,          -- 进程号
       t2.serial#,      -- 序列号
       t3.object_name,  -- 表名
       t2.OSUSER,       -- 操作系统用户名
       t2.MACHINE,      -- 机器名
       t2.PROGRAM,      -- 操作工具
       t2.LOGON_TIME,   -- 登录时间
       t2.COMMAND,
       t2.LOCKWAIT,     -- 表示当前这张表是否正在等待其他用户解锁这张表
       t2.SADDR,
       t2.PADDR,
       t2.TADDR,
       t2.SQL_ADDRESS,
       t1.LOCKED_MODE   -- 锁表模式
  from v$locked_object t1, v$session t2, dba_objects t3
 where t1.session_id = t2.sid
   and t1.object_id = t3.object_id
 order by t2.logon_time;

-- v$locked_object 视图中记录了所有session中的所有被锁定的对象信息。
-- v$session 视图记录了所有session的相关信息。
-- dba_objects 为oracle用户对象及系统对象的集合，通过关联这张表能够获取被锁定对象的详细信息。

--解除锁表，sid和seial#就是第一步中查询出来的进程号和序列号。
alter system kill session 'sid,seial#';
```
***
## 5. dmp导入导出
```bash
# 注意dmp默认不会导出空表,需执行下面sql查出的语句
# select 'alter table '||table_name||' allocate extent;' from user_tables where num_rows=0;
# full=y表示全导出，若无权限则只导出自己的
exp <username>/<password>@<ip>/<实例名> file=XXX.dmp full=y
# 导入使用的用户要具有dba权限
imp <username>/<password> file=XXX.dmp fromuser=<源库用户名> touser=<目标库用户名> ignore=y
```
***
## 6. 建表、建用户、分配权限
```sql
--(1)创建临时表空间(非必须)

--(2)创建表空间
create tablespace <tablespace_name> 
 datafile '/data/oracle_data/CSC2.dbf' size 1024M --存储地址 初始大小1G
 autoextend on next 10M maxsize unlimited   --每次扩展10M，无限制扩展
 EXTENT MANAGEMENT local  autoallocate  --说明表空间本地(local)管理，并自动分配范围(autoallocate)，用户不能指定范围的大小
 segment space management auto;  --段空间(segment)的空间管理上使用bitmaps(auto)来管理数据块。使用AUTO会比使用MANUAL有更好的空间利用率，与效能上的提升

 --(3)创建用户
 create user <username> identified by <password>
 default tablespace <tablespace_name>  --设置默认表空间
   temporary tablespace <temporary_tablespace_name> --设置临时表空间
   profile DEFAULT; --？

--(4)分配权限
   grant connect,resource to <username>;

  DBA: 拥有全部特权，是系统最高权限，只有DBA才可以创建数据库结构。
  RESOURCE:拥有Resource权限的用户只可以创建实体，不可以创建数据库结构。
  CONNECT:拥有Connect权限的用户只可以登录Oracle，不可以创建实体，不可以创建数据库结构。
  对于普通用户：授予connect, resource权限。
  对于DBA管理用户：授予connect，resource, dba权限。

  CONNECT角色： --是授予最终用户的典型权利，最基本的
        ALTER SESSION --修改会话
        CREATE CLUSTER --建立聚簇
        CREATE DATABASE LINK --建立数据库链接
        CREATE SEQUENCE --建立序列
        CREATE SESSION --建立会话
        CREATE SYNONYM --建立同义词
        CREATE VIEW --建立视图
    RESOURCE角色： --是授予开发人员的
        CREATE CLUSTER --建立聚簇
        CREATE PROCEDURE --建立过程
        CREATE SEQUENCE --建立序列
        CREATE TABLE --建表
        CREATE TRIGGER --建立触发器
        CREATE TYPE --建立类型
```
***
## 7. Oracle插入、删除字段

```sql
--添加字段的语法：
alter table tablename add (column datatype [default value][null/not null],….);
--修改字段的语法：
alter table tablename modify (column datatype [default value][null/not null],….);
--删除字段的语法：
alter table tablename drop (column);
```

------



# 三、oracle常用函数

## Oracle普通函数
### 1. nvl(v1,v2)
```sql
select nvl(e1,e2) from dual --e1不为空取e1,否则取e2
```
***
### 2. to_date(str,format) 

*时间转换函数：可将format格式的str转换成时间类型*

```sql
select to_date('2019-12-22 10:10:10', 'yyyy-mm-dd hh24:mi:ss') as test_date from dual;
--更新语句
update t_user set ACCOUNT_BEGIN_DATE = to_date('28-10-2019 10:10:10', 'dd-mm-yyyy hh24:mi:ss') ,ACCOUNT_END_DATE = to_date('2021-10-28 10:10:10', 'yyyy-mm-dd hh24:mi:ss') where 1 = 1;
-- 注：如果只有年月日可以使用 DATE+'yyyy-mm-dd' 这种方式
select DATE '2012-12-12' from dual;
```
***
### 3. add_months(date,n) 

*给date加（负数表示减）n月*

```sql
select add_months(sysdate,3) from dual; --sysdate表示当前时间，语句表示获取当前时间加上3个月后的时间

update cscgplx.t_overseastudentmain tov set tov.fundendtime = add_months(tov.arrivedate,(tov.imburseduration + nvl(tov.tempdelaymonth,'0')))
```
***
### 4. length(ch) 

*获取ch长度*

```sql
select length('1234') len from dual; --输出4
```
***
### 5. to_char(ch) 

*将值转成字符串*

```sql
select TO_CHAR(sysdate,'yyyy-mm-dd') from dual; --sysdate表示系统当前时间，将时间转成特定格式的字符串
select TO_CHAR(123) from dual; --输出字符串格式的123
```
***
### 6. sys_guid() 

*生成32位uuid*

```sql
select sys_guid() from dual; --32位uuid大写
select lower(sys_guid()) from dual; --32位uuid小写
```
***
### 7. substr(str, len)

*截取字符串，len>0表示从左截取，len<0表示从右截取*

```sql
select substr('aaabbb',3) from dual;
```

------

### 8. concat(str1, str2)

*拼接字符串，***只支持2个参数**

```sql
select concat('a','b') from dual;
select 'a'||'b' from dual; --这种方式也可以拼接字符串
```

------

## Oracle聚合函数

### 1. count() 

*统计数量*（**注：null不计入数量**）

>count(字段) 统计字段值**非空**的数量
>
>count( * )和count(1)统计记录数，应该使用count(*),sql会优化的！
```sql
select count(null) from dual; -- null不计入数量，返回0
```
### 2. sum(v) 

*将非空的v加到一起，忽略空值*

```sql
--指定用户id中获取没有受理单位工作人员角色的用户id
--having 类似group by后的where，但可接聚合函数，聚合组内元素
SELECT u.ID FROM T_USER u 
	LEFT JOIN T_USER_ROLE_MAP m ON m.USER_ID = u.ID 
    LEFT JOIN T_ROLE r ON m.ROLE_ID = r.ID 
WHERE u.DELETE_FLAG = '0' AND r.DELETE_FLAG = '0' AND u.ID in (?)
	GROUP BY u.ID 
	HAVING count( CASE WHEN r.CODE IN ( 'accept-unit-project-leader-school' ) THEN 1 ELSE null END ) = 0 --对于计数来说count性能优于sum
```
### 3. wmsys.wm_concat(v) 

*拼接函数:将v拼接到一起并以“,”分隔*

不推荐使用，wm_concat和distinct、to_char或group使用可能出现ORA-22922: 不存在的 LOB 值解决办法，推荐使用listagg函数

注意：wm_concat聚合的字段是**无序**的，哪怕你之前排序了也没用

```sql
--复杂sql示例：
SELECT PROJECT_NUM,
       PROJECT_NAME,
       NAME,
       CNAME,
       TOTAL_DESIGNATION,
       GROUP_UNITS,
       PROJECT_APPLICATION_TYPE,
       DECLARE_BATCH_NAME,
       IS_FUND,
       wmsys.wm_concat(DISTINCT D2_CNAME), --DISTINCT保证拼接后的结果不会重复
       wmsys.wm_concat(DISTINCT DA_CNAME),
       wmsys.wm_concat(DISTINCT UNIT_CNAME)
  FROM (SELECT t.PROJECT_NUM,
               t.PROJECT_NAME,
               o.NAME,
               t.PROJECT_APPLICATION_TYPE,
               d1.CNAME,
               t.TOTAL_DESIGNATION,
               t.GROUP_UNITS,
               t.DECLARE_BATCH_NAME,
               a.SUBJECTS_NAME,
               d2.CNAME AS D2_CNAME,
               da.CNAME AS DA_CNAME,
               u.UNIT_CNAME AS UNIT_CNAME,
               CASE (SELECT COUNT(*)
                   FROM PD_PROJECT_FUND
                  WHERE f.PROJECT_DECLARE_ID = t.ID)
                 WHEN 0 THEN
                  '否'
                 ELSE
                  '是'
               END AS IS_FUND
          FROM PD_DECLARE_INFO t
         INNER JOIN DICT_DATA d1
            ON d1.CODE_ID = t.SPECIAL_TYPE_ID
         INNER JOIN T_ORG o
            ON o.ID = t.LEAD_UNIT_ID
         INNER JOIN PD_DECLARE_SUBJECTS a
            ON a.PROJECT_DECLARE_ID = t.ID
         INNER JOIN PD_PROJECT_UNIT u
            ON u.PROJECT_DECLARE_ID = t.ID
         INNER JOIN PD_PROJECT_FUND f
            ON f.PROJECT_DECLARE_ID = t.ID
         INNER JOIN PD_PROJECT_IDENTITY i
            ON i.PROJECT_DECLARE_ID = t.ID
         INNER JOIN DICT_SPE_AREA da
            ON da.CODE_ID = u.COUNTRY_ID
         INNER JOIN DICT_DATA d2
            ON d2.CODE_ID = i.IDENTITY_TYPE
         WHERE t.DELETE_FLAG = '0'
           AND t.PROJECT_STATE = '7') mm
 GROUP BY PROJECT_NUM,
          PROJECT_NAME,
          NAME,
          CNAME,
          TOTAL_DESIGNATION,
          GROUP_UNITS,
          PROJECT_APPLICATION_TYPE,
          DECLARE_BATCH_NAME,
          IS_FUND;
```
### 4. listagg

wm_concat的升级版，Oracle官方推荐使用，将列以行的形式排列，可设置分隔关键字，及按字段排序

```sql
--listagg(XXX, ',') within group(order by YYY) 将XXX以,分割并按YYY聚合排序
SELECT t.advancebatchid,
       (SELECT BATCHNAME
          FROM t_scholarshipadvance_batch
         WHERE advancebatchid = t.advancebatchid) batchname,
       t.batchyear,
       t.batchmonth,
       (SELECT PEOPLENUMBER
          FROM t_scholarshipadvance_batch
         WHERE advancebatchid = t.advancebatchid) usercount,
       listagg(t.CURRENTAMOUNT || t.CURRENCY || t.PERSONNUM || '人', ',') within group(order by t.CURRENCY, t.CURRENTAMOUNT, t.PERSONNUM) currentAndAmountAndNum
  FROM (SELECT batch.advancebatchid,
               batch.batchyear,
               batch.batchmonth,
               release.CURRENCY,
               sum(release.CURRENTAMOUNT) CURRENTAMOUNT,
               count(release.studentid) as personnum
          FROM t_scholarshipadvance_batch batch
          LEFT JOIN t_scholarshipadvance_batch child
            ON child.SUPERIORBATCHID = batch.ADVANCEBATCHID
          LEFT JOIN t_scholarshipadvance_releasere release
            ON child.advancebatchid = release.advancebatchid
         WHERE EXISTS (SELECT 1
                  FROM t_advance_auditflowrecord taa
                 WHERE taa.auditnode = '4'
                   AND taa.advancebatchid = batch.advancebatchid)
           AND batch.BATCHNUM IS NOT NULL
           AND batch.batchstate = 6
           AND batch.ifueserful = 1
           AND batch.IFMERGEBATCH = '1'
           AND batch.SUPERIORBATCHID IS NULL
         GROUP BY batch.advancebatchid,
                  batch.batchyear,
                  batch.batchmonth,
                  release.CURRENCY
        UNION
        SELECT batch.advancebatchid,
               batch.batchyear,
               batch.batchmonth,
               release.CURRENCY,
               sum(release.CURRENTAMOUNT) CURRENTAMOUNT,
               count(release.studentid) as personnum
          FROM t_scholarshipadvance_batch batch
          LEFT JOIN t_scholarshipadvance_releasere release
            ON batch.advancebatchid = release.advancebatchid
         WHERE EXISTS (SELECT 1
                  FROM t_advance_auditflowrecord taa
                 WHERE taa.auditnode = '4'
                   AND taa.advancebatchid = batch.advancebatchid)
           AND batch.BATCHNUM IS NOT NULL
           AND batch.batchstate = 6
           AND batch.ifueserful = 1
           AND batch.IFMERGEBATCH = '0'
           AND batch.SUPERIORBATCHID IS NULL
         GROUP BY batch.advancebatchid,
                  batch.batchyear,
                  batch.batchmonth,
                  release.CURRENCY) t
 where t.batchyear = ?
   and t.batchmonth = ?
 GROUP BY t.advancebatchid, t.batchyear, t.batchmonth
 ORDER BY t.batchyear desc, t.batchmonth desc
```

------

# 四、sql优化（关系型数据库通用）

## 1. join、in、exists 效率比较
在sql中，join /in /exists 都可以用来实现，“**查询A表中在（或者不在）B表中的记录**”，这种查询，在查询的两个表大小相当的情况下，3种查询方式的执行时间通常是：
**exists <= in <= join** 
当表中字段允许NULL时，not in 的方式最慢；
**not exists <= left join <= not in**
关键：**in接小表 exists接大表**

```sql
--使用inner join 查询要1.1s左右
select *
  from tbl_abroadproject t -- 3900
 inner join (SELECT PROJECTID
               FROM TBL_AGENTCATPROJECT
              WHERE AGENTID in (select ae.agentid
                                  from tbl_agent_embassy ae
                                  left join (select orgid
                                              from TBL_GENERALUSER g
                                             group by g.orgid) b
                                    on ae.embassyid = b.orgid
                                 inner join TBL_GENERALUSER t
                                    on ae.embassyid = t.orgid
                                 where t.usertype = 1)) temp --7900
    on t.projectid = temp.PROJECTID;
--使用exists 查询只要 0.3秒左右
select *
  from tbl_abroadproject t --3900
 where EXISTS (SELECT 1
          FROM TBL_AGENTCATPROJECT
         WHERE t.projectid = PROJECTID
           and AGENTID in (select ae.agentid
                             from tbl_agent_embassy ae
                             left join (select orgid
                                         from TBL_GENERALUSER g
                                        group by g.orgid) b
                               on ae.embassyid = b.orgid
                            inner join TBL_GENERALUSER t
                               on ae.embassyid = t.orgid
                            where t.usertype = 1));
--继续优化inner并去除无用的left join,查询只要0.1s左右
select *
  from tbl_abroadproject ap --3900
 where EXISTS (SELECT 1 --select子句共7900
          FROM TBL_AGENTCATPROJECT acp --95849
         WHERE ap.projectid = acp.PROJECTID
           and acp.AGENTID in
               (select ae.agentid --select子句共44条，44<<95849,所以用in
                  from tbl_agent_embassy ae --44
                 where exists (select 1
                          from TBL_GENERALUSER t --1561
                         where ae.embassyid = t.orgid
                           and t.usertype = 1)));

```
## 2. left、right、inner  优化
### （1）优化原则

> **小表驱动大表，条件减少两表，减少返回数据，被驱动表关联条件建立索引。**
>
> `因为可减少驱动表循环，且被驱动表连接条件可使用索引（主要适用于nested-loop算法）`
>
> **小技巧：可通过select (select a from A where A.id=B.id)... from B这种方式避免大量left join，具体性能要看实际场景** 
> `left join 左表为驱动表，右表为被驱动表`
> `right join 右表为驱动表，左表为被驱动表`
> `inner join 数据量小的表为被驱动表`  
>
> 已下面的sql为例1：
>
> ```sql
> select * from evaluation e -- 注意在oracle中select的字段很重要，会影响是否使用索引！
>     left join association a
>       on e.association = a.code
>      and a.id = '3' -- a.id索引生效
>    where e.evaluationcode = '13150006' -- e.evaluationcode索引生效
> -----------------------------------------------------------------------------------------
> | Id  | Operation                      | Name          | Rows | Bytes | Cost | Time     |
> -----------------------------------------------------------------------------------------
> |   0 | SELECT STATEMENT               |               |    1 |   358 |    3 | 00:00:01 |
> |   1 |   NESTED LOOPS OUTER           |               |    1 |   358 |    3 | 00:00:01 |
> |   2 |    TABLE ACCESS BY INDEX ROWID | EVALUATION    |    1 |   320 |    2 | 00:00:01 |
> | * 3 |     INDEX RANGE SCAN           | IDX_EVA_CODE  |    1 |       |    1 | 00:00:01 |
> | * 4 |    TABLE ACCESS BY INDEX ROWID | ASSOCIATION   |    1 |    38 |    1 | 00:00:01 |
> | * 5 |     INDEX UNIQUE SCAN          | ASSOCIATIONID |    1 |       |    0 | 00:00:01 |
> -----------------------------------------------------------------------------------------
> ```
>
> 不管是什么数据库首先**优化器都会尽可能过滤表的数据**，所以会先执行驱动表条件e.evaluationcode = '13150006'和被驱动表条件a.id= '3'减小表数据，所以会走e.evaluationcode和a.id上的索引
>
> mysql8以下版本 会使用被驱动表a.id上的索引，注意不会使用驱动表e.association上的索引！，因为**mysql8以下版本使用的是nested loops算法**，循环驱动表数据，如果**被驱动表关联字段没有索引使用的是Block Nested-Loop Join，使用索引则使用 Index Nested-Loop Join算法**；mysql8以上根据优化器一般使用hash join。
>
> 由于是等值关联，如果没有条件，Oracle根据CBO优化器一般使用hash outer join，不会使用a.code上的索引；但由于a.id上有索引，Oracle优化器会使用Nested-Loop。
>
> **结论**：join使用什么连接算法取决于具体情况，但不管那种连接方式，**小表驱动大表，条件减少两表，减少返回数据，被驱动表关联条件建立索引**，总是没错的。
>
> 例2：下面表已在**t_user.user_name**和**T_USER_ROLE_MAP.user_id**上建立索引
>
> ```sql
> --由于在基表条件和被关联表关联条件上设置了索引，优化器判断Nested Loop优于Hash Join
> SELECT *
>   FROM CSC3TEST.t_user u
>   LEFT JOIN CSC3TEST.T_USER_ROLE_MAP urm
>     ON u.id = urm.user_id
>   where u.user_name = 'sysadmin01';
> -----------------------------------------------------------------------------------------------
> | Id  | Operation                      | Name                | Rows | Bytes | Cost | Time     |
> -----------------------------------------------------------------------------------------------
> |   0 | SELECT STATEMENT               |                     |    1 |   313 |    4 | 00:00:01 |
> |   1 |   NESTED LOOPS OUTER           |                     |    1 |   313 |    4 | 00:00:01 |
> |   2 |    TABLE ACCESS BY INDEX ROWID | T_USER              |    1 |   204 |    2 | 00:00:01 |
> | * 3 |     INDEX RANGE SCAN           | T_USER_USERNAME_IDX |    1 |       |    1 | 00:00:01 |
> |   4 |    TABLE ACCESS BY INDEX ROWID | T_USER_ROLE_MAP     |    1 |   109 |    2 | 00:00:01 |
> | * 5 |     INDEX RANGE SCAN           | PK_T_USER_ROLE_MAP  |    1 |       |    1 | 00:00:01 |
> -----------------------------------------------------------------------------------------------
> --不设置任何条件下，即便被关联表关联条件上设置了索引，优化器判断Hash Join优于Nested Loop
> SELECT *
>   FROM CSC3TEST.t_user u
>   LEFT JOIN CSC3TEST.T_USER_ROLE_MAP urm
>     ON u.id = urm.user_id;
> -------------------------------------------------------------------------------------
> | Id  | Operation               | Name            | Rows | Bytes  | Cost | Time     |
> -------------------------------------------------------------------------------------
> |   0 | SELECT STATEMENT        |                 |  648 | 202824 |   15 | 00:00:01 |
> | * 1 |   HASH JOIN RIGHT OUTER |                 |  648 | 202824 |   15 | 00:00:01 |
> |   2 |    TABLE ACCESS FULL    | T_USER_ROLE_MAP |  459 |  50031 |    5 | 00:00:01 |
> |   3 |    TABLE ACCESS FULL    | T_USER          |  648 | 132192 |    9 | 00:00:01 |
> -------------------------------------------------------------------------------------
> --若返回字段在索引中，hash join也会使用索引
> SELECT u.id, urm.user_id
>     FROM CSC3TEST.t_user u
>     LEFT JOIN CSC3TEST.T_USER_ROLE_MAP urm
>       ON u.id = urm.user_id
> ---------------------------------------------------------------------------------------
> | Id  | Operation               | Name               | Rows | Bytes | Cost | Time     |
> ---------------------------------------------------------------------------------------
> |   0 | SELECT STATEMENT        |                    |  648 | 33048 |    7 | 00:00:01 |
> | * 1 |   HASH JOIN OUTER       |                    |  648 | 33048 |    7 | 00:00:01 |
> |   2 |    INDEX FAST FULL SCAN | PK_T_USER          |  648 | 11664 |    3 | 00:00:01 |
> |   3 |    INDEX FAST FULL SCAN | PK_T_USER_ROLE_MAP |  459 | 15147 |    3 | 00:00:01 |
> ---------------------------------------------------------------------------------------
> --若是只查基表，会使用被关联表关联条件的索引
> SELECT u.*
>   FROM CSC3TEST.t_user u
>   LEFT JOIN CSC3TEST.T_USER_ROLE_MAP urm
>     ON u.id = urm.user_id;
> ----------------------------------------------------------------------------------------
> | Id  | Operation               | Name               | Rows | Bytes  | Cost | Time     |
> ----------------------------------------------------------------------------------------
> |   0 | SELECT STATEMENT        |                    |  648 | 153576 |   13 | 00:00:01 |
> | * 1 |   HASH JOIN RIGHT OUTER |                    |  648 | 153576 |   13 | 00:00:01 |
> |   2 |    INDEX FAST FULL SCAN | PK_T_USER_ROLE_MAP |  459 |  15147 |    3 | 00:00:01 |
> |   3 |    TABLE ACCESS FULL    | T_USER             |  648 | 132192 |    9 | 00:00:01 |
> ----------------------------------------------------------------------------------------
> --数据量接近的话，in和exists效率相同
> SELECT u.*
>   FROM CSC3TEST.t_user u where u.id in (select urm.user_id from CSC3TEST.T_USER_ROLE_MAP urm);
> SELECT u.*
>   FROM CSC3TEST.t_user u where exists (select 1 from CSC3TEST.T_USER_ROLE_MAP urm where urm.user_id = u.id)
> ----------------------------------------------------------------------------------------
> | Id  | Operation               | Name               | Rows | Bytes  | Cost | Time     |
> ----------------------------------------------------------------------------------------
> |   0 | SELECT STATEMENT        |                    |  423 | 100251 |   13 | 00:00:01 |
> | * 1 |   HASH JOIN RIGHT SEMI  |                    |  423 | 100251 |   13 | 00:00:01 |
> |   2 |    INDEX FAST FULL SCAN | PK_T_USER_ROLE_MAP |  459 |  15147 |    3 | 00:00:01 |
> |   3 |    TABLE ACCESS FULL    | T_USER             |  648 | 132192 |    9 | 00:00:01 |
> ----------------------------------------------------------------------------------------
> --数据在索引中：覆盖索引
> SELECT u.id
>   FROM CSC3TEST.t_user u
>   LEFT JOIN CSC3TEST.T_USER_ROLE_MAP urm
>     ON u.id = urm.user_id;
> ---------------------------------------------------------------------------------------
> | Id  | Operation               | Name               | Rows | Bytes | Cost | Time     |
> ---------------------------------------------------------------------------------------
> |   0 | SELECT STATEMENT        |                    |  648 | 33048 |    7 | 00:00:01 |
> | * 1 |   HASH JOIN OUTER       |                    |  648 | 33048 |    7 | 00:00:01 |
> |   2 |    INDEX FAST FULL SCAN | PK_T_USER          |  648 | 11664 |    3 | 00:00:01 |
> |   3 |    INDEX FAST FULL SCAN | PK_T_USER_ROLE_MAP |  459 | 15147 |    3 | 00:00:01 |
> ---------------------------------------------------------------------------------------
> ```

### （2）基表、被关联表条件优化

> **基表、被关联表条件：**(以下只适用于left join，right join，full join，不适合inner join)
> **（1）left join where + 基表过滤条件：先对基表执行过滤，然后进行left join；
> （2）left join where + 被关联表过滤条件：先执行left join，然后执行过滤条件；
> （3）left join on + 基表过滤条件：满足过滤的left join，不满足的后面补null，然后两集合并一起；
> （4）left join on + 被关联表过滤条件：先执行过滤条件，然后执行left join；**
>
> **结论：**出于性能考虑，应该**尽可能先对表过滤使表的数据量减少**，所以**基表条件放where，被关联表条件放on**

### （3）表连接算法

> 参考：https://www.cnblogs.com/Dreamer-1/p/6076440.html

#### ①  HASH JOIN（哈希连接）

> $$
> O(n)\approx n （被驱动表）
> $$
>
> 哈希连接只适用于等值连接（即连接条件为  =  ），mysql8以上才支持hash join，**注意hash join不走两表关联字段索引**
>
> 首先优化器会将**小表作为驱动表**，**扫描驱动表，利用 join 的关联字段在内存中建立散列表**，然后扫描被驱动表，每读出一行数据，并从散列表中找到与之对应数据。它是大数据集连接操时的常用方式，适用于驱动表的数据量较小，可以放入内存的场景，它对于**没有索引的大表和并行查询**的场景下能够提供最好的性能。可惜它只适用于**等值连接**的场景。
>
> 等值连接在没有建立**关联字段索引**的话，默认会走hash join，若是索引设置的特别好，优化器也可能使用nested loop
>
> ![](https://pic3.zhimg.com/v2-7af3ee1a90f7d200e0c3c4456b25c9fa_b.jpg)

#### ②  NESTED LOOPS（嵌套循环）

> 内部连接过程：
> 		a) 取出 row source 1 的 row 1（第一行数据），遍历 row source 2 的所有行并检查是否有匹配的，取出匹配的行放入结果集中
> 		b) 取出 row source 1 的 row 2（第二行数据），遍历 row source 2 的所有行并检查是否有匹配的，取出匹配的行放入结果集中
> 		c) ……
> 	若 row source 1 （即驱动表）中返回了 N 行数据，则 row source 2 也相应的会被全表遍历 N 次。
> 	因为 row source 1 的每一行都会去匹配 row source 2 的所有行，所以当 row source 1 返回的行数尽可能少并且能高效访问 row source 2（如建立适当的索引）时，效率较高。
> 	延伸：
> 		嵌套循环的表有驱动顺序，注意选择合适的驱动表。
> 		嵌套循环连接有一个其他连接方式没有的好处是：可以先返回已经连接的行，而不必等所有的连接操作处理完才返回数据，这样可以实现快速响应。
> 		应尽可能使用限制条件（Where过滤条件）使驱动表（row source 1）返回的行数尽可能少，
> 		同时在匹配表（row source 2）的连接操作关联列上建立唯一索引（UNIQUE INDEX）或是选择性较好的非唯一索引，此时嵌套循环连接的执行效率会变得很高。
> 		若驱动表返回的行数较多，即使匹配表连接操作关联列上存在索引，连接效率也不会很高。

**mysql join算法** 

`参考`：https://zhuanlan.zhihu.com/p/54275505 https://blog.csdn.net/qq_27529917/article/details/87904179

```sql
--示例sql：
select * from user tb1 left join level tb2 on tb1.id = tb2.user_id;
```

- **Simple Nested-Loop Join（简单的嵌套循环连接）**

> $$
> O(m,n)\approx m\cdot n	（m表示驱动表，n表示被驱动表）
> $$
>
> **嵌套循环连接算法**就是一个**双层for 循环** ，通过**循环驱动表**的行数据，逐个与**被驱动表的所有行数据**进行比较来获取结果
>
> **mysql不会使用这种算法，而是根据情况使用下面两种算法**

![](https://pic1.zhimg.com/80/v2-2b9d48da48c6c436283fdec14db9d174_720w.jpg)

- **Index Nested-Loop Join（索引嵌套循环连接）**

  > $$
  > O(m,n)\approx m\cdot log(n)	（m表示驱动表，n表示被驱动表）
  > $$
  >
  > 减少**被驱动表**的匹配次数，**索引嵌套循环连接**是用**驱动表匹配条件来匹配被驱动表的索引**极大的减少了对内层表的匹配次数
  >
  > **匹配次数 = row(驱动表) * row(被驱动表) => row(外层表) * height(内层表索引)**

![](https://pic4.zhimg.com/80/v2-c8790aa879ca6fedb83d529558bb40e3_720w.jpg)

- **Block Nested-Loop Join（缓存块嵌套循环连接）**

> **缓存块嵌套循环连接算法**意在通过**一次性缓存外层表的多条数据**，以此来**减少内层表的扫表次数**，从而达到提升性能的目的。如果无法使用**Index Nested-Loop Join**的时候，mysql是默认使用的是**Block Nested-Loop Join算法的**。
>
> ```sql
> Show variables like 'join_buffer_size%'; -- 通过join_buffer_size参数可设置join buffer的大小
> ```
>
> 会把驱动表实际使用的列都放到join buffer里（查询或者条件里涉及的都会使用到），驱动表的扫描次数
>
> ```
> Scaninner_table = (RN * used_column_size) / join_buffer_size + 1
> ```
>
> 所以可以提前分配join_buffer_size，尽可能减小扫描次数，最好为1
>
> 当level 表的 user_id 不为索引的时候，默认会使用Block Nested-Loop Join算法，**注意会根据条件和select的字段将数据放到join buffer中（so要加条件和减少select的字段以增大join buffer缓存的数据）**，匹配的过程类似下图

![](https://pic3.zhimg.com/80/v2-0e81dd7fe538f67559bc24c0a5a3207e_720w.jpg)

- **mysql join总结**

  > 1. **永远用小结果集驱动大结果集，可通过where驱动表条件缩小结果集(其本质就是减少外层循环的数据数量)**
  > 2. **被驱动表关联条件要设置索引**
  > 3. **增大join buffer size的大小（一次缓存的数据越多，那么内层包的扫表次数就越少）**
  > 4. **减少不必要的字段查询（字段越少，join buffer 所缓存的数据就越多）**

#### ③  SORT MERGE JOIN

> ​	假设有查询：select a.name, b.name from table_A a join table_B b on (a.id = b.id)
> ​	内部连接过程：
> ​		a) 生成 row source 1 需要的数据，按照**连接操作关联列（如示例中的a.id）对这些数据进行排序**
> ​		b) 生成 row source 2 需要的数据，按照与 a) 中对应的**连接操作关联列（b.id）对数据进行排序**
> ​		c) **两边已排序的行放在一起执行合并操作**（对两边的数据集进行扫描并判断是否连接）
> ​	延伸：
> ​		如果示例中的连接操作**关联列 a.id，b.id 之前就已经被排过序**了的话，连接速度便可大大提高（**比hash join更快**），因为排序是很费时间和资源的操作，尤其对于有大量数据的表。
> ​		故可以考虑在 a.id，b.id 上建立索引让其能预先排好序。不过遗憾的是，由于返回的结果集中包括所有字段，所以通常的执行计划中，即使连接列存在索引，也不会进入到执行计划中，除非进行一些特定列处理（如仅仅只查询有索引的列等）。
> ​		排序-合并连接的表无驱动顺序，谁在前面都可以；
> ​		排序-合并连接适用的连接条件有： <   <=   =   >   >= ，不适用的连接条件有： <>    like

### （4）表连接类型

> INNER JOIN（内连接）<=> join
> OUTER JOIN（外连接）
> 	left outer join <=> **left join** #左外连接，**返回左表全部数据**
> 	right outer join <=> **right join** #右外连接，**返回右表全部数据**
> 	FULL OUTER JOIN<=> FULL JOIN #全外连接,返回左右两表的全部记录。(左右两边不匹配的项都以空值代替)

------

## 3. 数据库关键字执行顺序（通用）

>select distinct <select_list> 
>    from <left_table> <join_type> join <right_table> on <join_condition> 
>    where <where_condtion>
>    group by <group_by_list>
>    having <having_condition>
>
>    order by <order_by_condition>
>    limit <offset>,<count>
>①**From**：对from左边的表和右边的表计算笛卡尔积，产生虚拟表c1
>②**On**：对c1中的数据进行on过滤，只有符合过滤条件的数据记录才会记录在虚拟表c2中
>③**Join**：若指定了连接条件(left、right)，主表中的未匹配的行就会作为外部行添加到c2中，生成虚拟表c3
>④**Where**：对虚拟表c3中的数据进行条件过滤，符合过滤条件的记录插入到虚拟表c4中
>⑤**Group by**：根据group by子句中的列，对c4中的记录进行分组操作，生成c5
>⑥**Having**：对虚拟表c5中的记录进行having过滤，符合筛选条件的记录插入虚拟表c6中
>⑦**Select**：执行select操作，选择指定的列，插入到虚拟表c7中
>⑧**Distinct**：对c7中的数据去重，生成虚拟表c8
>⑨**Order by**：对虚拟表c8中的数据按照指定的排序规则进行排序，生成虚拟表c9 # 如果Distinct和Order by共用的话，Distinct的字段order by必须有，否则会报错
>⑩**Limit**：取出指定的记录，产生虚拟表c10，将结果返回
>
>
>-- 因为sql执行顺序where在group by之前，所以要对group by后的结果集进行过滤就需要使用having，having内可使用聚合函数，对每组组内的数据执行聚合，也可跟where一样使用
>
>order by 默认升序，可接多个参数，优先级按参数顺序，如：order by A ASC, B DESC;

# 五、复杂sql示例

```sql
insert into CSC3TEST.FANJC_AM_PRO_MAG_PERMISSION
  (ID,
   USER_ID,
   PROJECT_ID,
   PROJECT_NAME,
   CHANNEL_CODE,
   CHANNEL_NAME,
   COUNTRY_ID,
   EMBASSY_ID,
   COUNTRY_NAME,
   EMBASSY_NAME,
   ADMISSION_STATUS,
   PERMISSION_STATUS,
   DELETE_FLAG)
  select sys_guid(),
         u.USERID,
         ap.PARENTID as PROJECTID,
         (select tap.PROJECTNAME
            from CSCADMIN.tbl_abroadproject tap
           where tap.PROJECTID = ap.PARENTID) AS PROJECT_NAME,
         case ap.PARENTID
           when '-1' then
            '所有'
           else
            ap.PROJECTCODE
         END AS CHANNEL_CODE,
         ap.PROJECTNAME as CHANNEL_NAME,
         '所有' as COUNTRY_ID,
         '所有' as EMBASSY_ID,
         '所有' as COUNTRY_NAME,
         '所有' as EMBASSY_NAME,
         0 as ADMISSION_STATUS,
         '1' as PERMISSION_STATUS,
         '0' as DELETE_FLAG
    from CSCADMIN.TBL_AGENTCATPROJECT acp
   inner join (select b.USERID, ae.AGENTID
                 from CSCADMIN.tbl_agent_embassy ae
                 left join (select g.orgid, g.USERID
                             from CSCADMIN.TBL_GENERALUSER g
                            where g.USERTYPE = 1) b
                   on ae.embassyid = b.orgid) u
      on u.AGENTID = acp.AGENTID
   inner join CSCADMIN.tbl_abroadproject ap
      on ap.projectid = acp.PROJECTID
   where u.USERID in (SELECT tu.ID
                        FROM CSC3TEST.T_USER tu
                        LEFT JOIN CSC3TEST.T_USER_ROLE_MAP m
                          ON m.USER_ID = tu.ID
                        LEFT JOIN CSC3TEST.T_ROLE r
                          ON m.ROLE_ID = r.ID
                       WHERE tu.DELETE_FLAG = '0'
                         AND r.DELETE_FLAG = '0'
                         AND r.CODE = 'accept-unit-project-leader-school'
                         AND length(tu.id) < 32)

insert into CSC3TEST.FANJC_AM_PRO_MAG_PERMISSION
  (ID,
   USER_ID,
   PROJECT_ID,
   PROJECT_NAME,
   CHANNEL_CODE,
   CHANNEL_NAME,
   COUNTRY_ID,
   EMBASSY_ID,
   COUNTRY_NAME,
   EMBASSY_NAME,
   ADMISSION_STATUS,
   PERMISSION_STATUS,
   DELETE_FLAG)
  select sys_guid(),
         gu.USERID,
         ap.PARENTID as PROJECTID,
         (select tap.PROJECTNAME
            from CSCADMIN.tbl_abroadproject tap
           where tap.PROJECTID = ap.PARENTID) AS PROJECT_NAME,
         case ap.PARENTID
           when '-1' then
            '所有'
           else
            ap.PROJECTCODE
         END AS CHANNEL_CODE,
         ap.PROJECTNAME as CHANNEL_NAME,
         '所有' as COUNTRY_ID,
         '所有' as EMBASSY_ID,
         '所有' as COUNTRY_NAME,
         '所有' as EMBASSY_NAME,
         0 as ADMISSION_STATUS,
         '1' as PERMISSION_STATUS,
         '0' as DELETE_FLAG
    from CSCADMIN.TBL_AGENTUSERCATPROJECT aup
   inner join CSCADMIN.tbl_abroadproject ap
      on ap.PROJECTID = aup.PROJECTID
   inner join CSCADMIN.TBL_GENERALUSER gu
      on aup.auditorid = gu.userid
   where aup.auditorid in
         (SELECT tu.ID
            FROM CSC3TEST.T_USER tu
            LEFT JOIN CSC3TEST.T_USER_ROLE_MAP m
              ON m.USER_ID = tu.ID
            LEFT JOIN CSC3TEST.T_ROLE r
              ON m.ROLE_ID = r.ID
           WHERE tu.DELETE_FLAG = '0'
             AND r.DELETE_FLAG = '0'
             AND r.CODE = 'accept-unit-project-leader-school'
             AND length(tu.id) < 32
             and EXISTS (select t.userid
                    from CSCADMIN.TBL_GENERALUSER t
                   where t.usertype in (0, 99)
                     and tu.id = t.USERID));	   
```

