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

### mysql常用命令

```sql
#查看表结构
desc <table_name>;
```

mysql优化

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

