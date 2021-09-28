### mysql建表备忘

```mysql
#设置自动增长列的初始值
auto_incremenrt
#较全的建表语句
CREATE TABLE `table1` (
  `table1_id` int NOT NULL AUTO_INCREMENT,
  `table1_type` tinyint(1) NOT NULL DEFAULT '0' COMMENT '1类 2类 0其它',
  `table1_no` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL COMMENT '编码，必须唯一，可以系统生成',
  `table1_name` varchar(255) COLLATE utf8mb4_general_ci NOT NULL COMMENT '名称',
  `short_name` varchar(255) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '简称',
  `alias` varchar(255) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '别名',
  `spec_count` int NOT NULL DEFAULT '0' COMMENT '多规格个数',
  `class_id` int NOT NULL DEFAULT '0' COMMENT '分类id,0表示无分类',
  `brand_id` int NOT NULL DEFAULT '0' COMMENT '品牌ID',
  `unit` smallint NOT NULL DEFAULT '0' COMMENT '基本单位',
  `aux_unit` smallint NOT NULL DEFAULT '0' COMMENT '辅助单位',
  `pinyin` varchar(40) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
  `origin` varchar(64) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '产地',
  `remark` varchar(512) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
  `seller_id` int NOT NULL DEFAULT '-1' COMMENT '人员id',
  `currency` varchar(40) COLLATE utf8mb4_general_ci NOT NULL DEFAULT 'CNY' COMMENT '币种',
  `flag_id` smallint NOT NULL DEFAULT '0' COMMENT '标记',
  `properties` varchar(1024) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '0,0,0,0,0,0',
  `prop1` varchar(100) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '自定义属性1',
  `prop2` varchar(100) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '自定义属性2',
  `prop3` varchar(100) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '自定义属性3',
  `deleted` int NOT NULL DEFAULT '0' COMMENT '删除时间',
  `version_id` int NOT NULL DEFAULT '0' COMMENT '版本号，用来检查同时修改的',
  `modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `created` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`table1_id`),                         # 主键
  UNIQUE KEY `UK_table1_no` (`table1_no`,`deleted`), # 唯一索引
  KEY `FK_table1_class_id` (`class_id`),             # 常规索引
  KEY `FK_table1_brand_id` (`brand_id`) USING BTREE, # 索引使用btree
  KEY `IX_table1_type` (`goods_type`) ,
  KEY `IX_table1_modified` (`modified`),
  # 外键，若插入时table1_brand没有brand_id对应数据则报外键错误
  CONSTRAINT `FK_table1_brand_id` FOREIGN KEY (`brand_id`) REFERENCES `table1_brand` (`brand_id`), 
  CONSTRAINT `FK_table1_class_id` FOREIGN KEY (`class_id`) REFERENCES `table1_class` (`class_id`)
) ENGINE=InnoDB AUTO_INCREMENT=100 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='XX表';
```

### mysql字段类型备忘

```mysql
tinyint #占一个字节(8bit)
```

### mysql json字段

```mysql
mysql json字段获取,支持json和数组
#category={'id':"XXX",name:"XXX"} tags=[1,3,5]
> mysql> SELECT id, category->'$.id', category->'$.name', tags->'$[0]', tags->'$[2]' FROM lnmp;
> +----+------------------+--------------------+--------------+--------------+
> | id | category->'$.id' | category->'$.name' | tags->'$[0]' | tags->'$[2]' |
> +----+------------------+--------------------+--------------+--------------+
> |  1 | 1                | "lnmp.cn"          | 1            | 3            |
> |  2 | 2                | "php.net"          | 1            | 5            |
> +----+------------------+--------------------+--------------+--------------+
> 2 rows in set (0.00 sec)</pre>

update cfg_right_group
set functional_rights = JSON_SET(functional_rights,'$.order_processing',json_merge(functional_rights->>'$.order_processing','"btn_notmatched_export_data"'))
where type = 0 and locate('btn_or_notmatched',functional_rights) and locate('btn_op_export_data',functional_rights) and group_id = 655
and locate('btn_notmatched_look_data',functional_rights);

#给json字段functional_rights内的order_processing数组添加btn_notmatched_export_data，返回order_processing数组
select json_merge(functional_rights->>'$.order_processing','"btn_notmatched_export_data"') as a from cfg_right_group 
where type = 0 and locate('btn_or_notmatched',functional_rights) and group_id = 655
and locate('btn_notmatched_look_data',functional_rights);

#给json字段functional_rights内的order_processing数组添加btn_notmatched_export_data，
#并替换原order_processing，返回functional_rights
select JSON_SET(functional_rights,'$.order_processing',json_merge(functional_rights->>'$.order_processing','"btn_notmatched_export_data"')) as a from cfg_right_group 
where type = 0 and locate('btn_or_notmatched',functional_rights) and group_id = 655
and locate('btn_notmatched_look_data',functional_rights);
```

### mysql默认排序规则

mysql in 查出的顺序与in内字段的顺序无关，这个问题其实等效于mysql默认查询排序方式

mysql查询默认排序将取决于查询中使用的索引以及它们的使用顺序，它可以随着数据/统计数据的变化以及优化器选择不同的计划而变化。但永远不要依赖任何数据库中的默认或隐式排序，而是应该使用order by排序。

```sql
#id为自增主键，下面的sql返回的顺序却是1，2，3，
select * from table1 where id in (3,1,2)
```

如果还想要按照in内顺序来排序，可使用下面的sql，

```sql
SELECT * FROM table1 WHERE id IN (3,1,2) order by FIELD(id, 3,1,2);
```

### mysql常用函数

```mysql
#length(s)-获取s长度
select length('abc');  #return 3
#trim(s)---s去除前后空格
select trim(' aa a '); #return aa a
#concat(str1,str2,...)---拼接字符串
select concat('aa','-','bb'); # return aa-bb
#find_in_set(str,strlist)
##str 要查询的字符串
##strList 字段名，参数以“,”分隔，如(1,2,6,8)
##查询字段(strList)中包含的结果，返回结果null或记录。
select find_in_set('b','a,b,c,d,e,f'); #return b
```

### mysql常用命令

```sql
#查看表结构
desc <table_name>;

---事务操作---
#1.开启事务
begin;
#2.执行sql
...
#3.提交/回滚事务
commit;
rollback;
```

mysql进阶命令

```mysql
#可查看innodb状态，例如可查看锁情况
show engine innodb status;
#显示用户正在运行的线程
show processlist;
#查看被锁的表
show open tables where In_use>0;
```

### mysql常用语句备忘

```mysql
#删除重复数据
--1.删除重复未启用的物理商，保留id最大的
delete s1 from sys_logistics_provider s1 inner join sys_logistics_provider s2
where s1.logistics_provider_id = s2.logistics_provider_id and (s1.is_active = 0 and s1.rec_id < s2.rec_id);
--2.删除同时存在启用与未启用的物理商中未启用的
delete s1 from sys_logistics_provider s1 inner join sys_logistics_provider s2
where s1.logistics_provider_id = s2.logistics_provider_id and (s1.is_active <> s2.is_active and s1.is_active = 0);

#复制表，将表sourceTable的结构及数据创建并复制到targetTable中
--方式1
CREATE TABLE targetTable LIKE sourceTable;
INSERT INTO targetTable SELECT * FROM sourceTable;
--方式2，更简洁
CREATE TABLE targetTable AS (SELECT * FROM sourceTable);
```

### mysql  duplicate操作

```mysql
#不存在唯一索引或主键冲突则插入，否则更新（注意，要使用这条语句，前提条件是这个表必须有一个唯一索引或主键）
#详细解释：若insert存在唯一索引或主键冲突则更新update后的值，否则插入values内的值
insert into `table` (`f1`, `f2`,`f3`,`f4`) values (v1, v2, v3, v4) on duplicate key update `f3` = v33, `f4` = v44
#不存在唯一索引或主键冲突则插入，否则忽略
insert ignore into `table` (`f1`, `f2`,`f3`,`f4`) values (v1, v2, v3, v4)
```

### mysql位运算

```sql
select 5 | 8; #位或，二进制位有1就1，否则为0
select 5 & 8; #位与，二进制位都为1，则为1，否则为0
select 5 ^ 8; #位异或，二进制位不同为1，否则为0
select ~5;    #位取反，相当于取补码
SELECT BIN(5);#查看数字二进制
```

### mysql唯一索引坑

假设唯一索引设置的字段可为Null，则多个null是不会冲突的

在mysql 的innodb引擎中，是允许在唯一索引的字段中出现多个null值的。

根据NULL的定义，NULL表示的是未知，因此两个NULL比较的结果既不相等，也不不等，结果仍然是未知。根据这个定义，多个NULL值的存在应该不违反唯一约束，所以是合理的，在oracel也是如此。

### mysql优化

optimizer_trace可查看实际sql的执行，有时间研究一下

```sql
EXPLAIN SELECT
	`goods_spec`.`spec_id`,
	`goods_spec`.`goods_id`,
	`goods_spec`.`spec_no`,
	`goods_spec`.`spec_code`,
	`goods_spec`.`barcode`,
	`goods_spec`.`spec_name`,
	`goods_spec`.`is_allow_neg_stock`,
	`goods_spec`.`is_not_need_examine`,
	`goods_spec`.`is_sn_enable`,
	`goods_spec`.`is_hotcake`,
	`goods_spec`.`is_allow_zero_cost`,
	`goods_spec`.`is_allow_lower_cost`,
	`goods_spec`.`retail_price`,
	`goods_spec`.`sale_score`,
	`goods_spec`.`pack_score`,
	`goods_spec`.`pick_score`,
	`goods_spec`.`validity_days`,
	`goods_spec`.`receive_days`,
	`goods_spec`.`weight`,
	`goods_spec`.`length`,
	`goods_spec`.`width`,
	`goods_spec`.`height`,
	`goods_spec`.`washing_label`,
	`goods_spec`.`tax_rate`,
	`goods_spec`.`large_type`,
	`goods_spec`.`unit`,
	`goods_spec`.`aux_unit`,
	`goods_spec`.`declare_name_cn`,
	`goods_spec`.`declare_name_en`,
	`goods_spec`.`declare_weight`,
	`goods_spec`.`declare_price`,
	`goods_spec`.`hs_code`,
	`goods_spec`.`goods_property`,
	`goods_spec`.`with_battery`,
	`goods_spec`.`with_magnet`,
	`goods_spec`.`is_powder`,
	`goods_spec`.`is_liquid`,
	`goods_spec`.`is_cosmetic`,
	`goods_spec`.`is_solid`,
	`goods_spec`.`is_paste`,
	`goods_spec`.`is_danger`,
	`goods_spec`.`is_metal`,
	`goods_spec`.`ali_properties`,
	`goods_spec`.`flag_id`,
	`goods_spec`.`img_url`,
	`goods_spec`.`img_key`,
	`goods_spec`.`barcode_count`,
	`goods_spec`.`plat_spec_count`,
	`goods_spec`.`remark`,
	`goods_spec`.`deleted`,
	`goods_spec`.`prop1`,
	`goods_spec`.`prop2`,
	`goods_spec`.`prop3`,
	`goods_spec`.`prop4`,
	`goods_spec`.`prop5`,
	`goods_spec`.`prop6`,
	`goods_spec`.`cost`,
	`goods_spec`.`version_id`,
	`goods_spec`.`modified`,
	`goods_spec`.`created`,
	`goods_goods`.`currency`,
	ifnull( cast( `goods_spec`.`cost` AS CHAR ), '成本缺失' ) AS `cost_str` 
FROM
	`goods_spec`
	LEFT OUTER JOIN `goods_goods` ON `goods_spec`.`goods_id` = `goods_goods`.`goods_id`
	JOIN (
	SELECT
		`goods_spec`.`spec_id` 
	FROM
		`goods_spec`
		LEFT OUTER JOIN `goods_goods` ON `goods_spec`.`goods_id` = `goods_goods`.`goods_id` 
	WHERE
		( `goods_spec`.`deleted` = 0 AND `goods_spec`.`spec_no` = 'XXX' ) 
	GROUP BY
		`goods_spec`.`spec_id` 
	ORDER BY
		`goods_spec`.`spec_id` DESC 
		LIMIT 50 
	) AS `alias_85486322` USING ( `spec_id` ) 
ORDER BY
	`goods_spec`.`spec_id` DESC
```

### INNER JOIN vs “FROM”中的多个表名

下面两种方式等价，但更推荐内连接方式，因为可读性更好，并且from多个表名还容易漏条件

```sql
#隐式连接
SELECT
    [a].[NoticeText]    
FROM
    [dbo].[Notice] [a],
    [dbo].[User] [b],
    [dbo].[WechatSE] [c]
WHERE
    [a].[UserID] =[b].[UserID]
    AND [b].[UserID]=[c].[UserID]
    AND [c].[OpenID]=@OpenID;
#内连接方式
SELECT [a].[NoticeText]     
    FROM [dbo].[Notice] [a]
        INNER JOIN [dbo].[User] [b] ON [a].[UserID] =[b].[UserID]
        INNER JOIN [dbo].[WechatSE] [c] ON [b].[UserID]=[c].[UserID]
    WHERE [c].[OpenID]=@OpenID;
```

mysql支持联表update和delete，但貌似不推荐使用

优化实践：

gs和gg平均各100万数据，都使用innodb引擎，mysql8

gg:gs <=> 1:*

goods_id 和 spec_id为各自主键，gs.spec_no有B-tree索引

```sql
#优化前sql,执行要20多s
SELECT *
FROM gs LEFT OUTER
JOIN gg
    ON gs.goods_id = gg.goods_id
GROUP BY  gs.spec_id #需要注意的是去掉该行后，只需要几十-几百毫秒（别问我为啥有Group by，问就是历史问题）
ORDER BY  gs.spec_no ASC LIMIT 50;
```

执行计划

| select_type | table | type   | key     | ref          | rows    | extre                           |
| ----------- | ----- | ------ | ------- | ------------ | ------- | ------------------------------- |
| SIMPLE      | gs    | index  | PRIMARY | null         | 1188408 | Using temporary; Using filesort |
| SIMPLE      | gg    | eq_ref | PRIMARY | gg..goods_id | 1       |                                 |

若去掉存在group by，执行计划如下

| select_type | table | type   | key              | ref          | rows | extre |
| ----------- | ----- | ------ | ---------------- | ------------ | ---- | ----- |
| SIMPLE      | gs    | index  | IX_goods_spec_no | null         | 50   | null  |
| SIMPLE      | gg    | eq_ref | PRIMARY          | gg..goods_id | 1    | null  |

可见**存在联表，若group by和order by字段不同，会对结果集进行全表排序（即便排序字段有索引），再分页**；

**相同或省略group by时，若order by字段存在索引(并且语句也走了该索引)，则不会再排序（因为btree本身就是有序的）,并且分页只扫描limit所限制的数字**；

**这里联表很关键，若不连表，也只需几十毫秒（GROUP取了个巧，默认就是主键排序的，换成非索引单表照样全表排序），说明联表后数据集巨大**

**根本原因：联表情况下，优化器可能会选择非order by字段的索引，导致会对联表结果集重新排序(若有where条件会限制的结果集大小)，目前情况就是导致了联表后100w数量级的重排序，必然会很慢**

优化方式：采用临时表方式，先缩短主表数量后，再左连接附表

```sql
#优化后sql,只需几十-几百毫秒
SELECT t.spec_id
FROM 
    (SELECT gs.spec_id,
        gs.goods_id
    FROM gs
    GROUP BY  gs.spec_id #默认就是按主键的，这个其实没什么用
    ORDER BY  gs.spec_no ASC LIMIT 50) t LEFT OUTER
JOIN gg
    ON t.goods_id = gg.goods_id;
ORDER BY  t.spec_no ASC; 
```

### mysql select字段数及类型对查询效率的影响

先抛出情况

```sql
#① table1数量165000左右，该sql查出1w左右条数据，查询均相同且走索引，*或者查询大部分字段需要10s左右
select * from table1 where ...
#② 查询varchar(255)字段3-4s左右
select varchar_255 from table1 where ...
#③ 查询数字，不到1s
select int_1 from table1 where ...
```

explain分析sql的执行计划是相同的，通过navicat分析sql状态，

唯一明显的差别在于Bytes_sent（The number of bytes sent from server.）

1.  查全部     5773978
2.  查字符串 1277638
3.  查数字     74192

可见**查询不同的类型，返回的数据大小是有显著差异的**（字符串>>数字）及**查询部分还是所有的字段** 都**很大程度影响了sql的效率**，所以代码规范上有select尽量只差所需要的字段，避免查多余的字段
