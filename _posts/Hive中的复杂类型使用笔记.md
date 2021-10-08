---
title: Hive中的复杂类型使用笔记
tags: 大数据
categories: 大数据
abbrlink: dc49c8ae
date: 2021-09-14 15:23:51
---

Hive作为一个SQL接口的分析引擎，也支持复杂类型，像Map，Array等，这些知识点不复杂，日常也会比较常用，作为零散笔记记录一下。

## Map数据结构

```
--创建
create table temp_db.map_test(
   	id int comment "源数据主键id",
	smap map<string,string> comment "string型map",
	imap map<string,int> comment "int型map"
);

--插入
insert into temp_db.map_test(id,smap,imap) select 12,map('姓名','张三') as pp,map('年龄',23,'性别',1) as dd;
insert into temp_db.map_test(id,smap,imap)select 14,map('地址','安徽') as dd,map('年级',3);

--注意，这里的map引用使用"()",有时候会错误写成"{}";此外，对于key-value值来说，是没有特定的限制的。key可以有多个。如上"姓名","地址"

--查询
select smap['姓名'] as arg1,imap['年龄'] as age from temp_db.map_test;

--key键查询
--map_keys(colName)  结果是一个Array,如果希望提取，则使用[index],如map_keys(smap)[0]
--hive和prest的index起点存在差异,hive从0开始，presto从1开始【我测试的环境是这样的】
select map_keys(smap) as smap_keys,map_keys(imap) as imap_keys from temp_db.map_test;


--value值查询
--map_values(colname)
select map_values(smap) as s_values,map_values(imap) as i_values from temp_db.map_test;


--键值对查询
--size(colName),返回对应列有多少个key-value
select size(imap) as pair_cnt from temp_db.map_test;

--map类型数据的加工，将map列拆分为key、value列

-- smap中只存在单个key-value的情况，所有lateral之后，数据有单列变成双列。但是行数没有变化
select id,skey,svalue from temp_db.map_test lateral view explode(smap) tb as skey,svalue;

-- imap中 存在多个键值对。这顿操作之后，行数会增加
select id,ikey,ivalue from temp_db.map_test lateral view explode(imap) tb as ikey,ivalue;
```

## Array数据结构

```
--创建
create table temp_db.array_test(
	id int comment '源数据主键id',
	year_arr array<string> comment '数组记录，年份',
	score_arr array<string> comment '数组记录，分数'
);

--插入
insert into  temp_db.array_test (id,year_arr,score_arr) select 12,array('1991','1990','1989'),array('56','20','23')

--查询
-- 注意事项，如果数组越界了，则报错。
select id,year_arr[1],year_arr[2] from temp_db.array_test

--是否包含某个值(array_contains()),Boolean型(true/false，where条件中比较合适)
select * from temp_db.array_test where array_contains(year_arr,'1990');

--拆成单条多行记录
select col1 from temp_db.array_test lateral view explode(year_arr) tb as col1
```

## Json数据结构
```
--创建
create table temp_db.json_test
(id int comment '源数据库id主键',
 str string comment '日志字符串');

--插入
insert into temp_db.json_test(id,str)
values (1,'{"name":"孙先生","carrer":"大数据开发工程师","dream":["开个便利店","去外面逛一逛","看本好书"],"friend":{
       "friend_1":"MM",
       "friend_2":"NN",
       "friend_3":"BB",
       "friend_4":"VV"
       }
        }');
insert into temp_db.json_test(id,str)
values (2,'{"name":"唐女士","carrer":"退休农民","dream":["儿子听话","带孙子"],"friend":{
       "friend_1":"CC"
       }
      }');

--json_tuple提取数据
-- 提取一级格式下的数据
select name 
from temp_db.json_test 
lateral view json_tuple(str,'name') tb as name;

-- 提取二级格式下的数据(如好友1)
select good_friend_1
from temp_db.json_test
lateral view json_tuple(str,'friend') dd as good_friend
lateral view json_tuple(good_friend,'好友1') tb as good_friend_1;

-- 提取标签中所有的内容(没有的标签，返回null)
select good_friend_1,good_friend_2,good_friend_3
from temp_db.json_test
lateral view json_tuple(str,'friend') dd as good_friend
lateral view json_tuple(good_friend,'好友1','好友2','好友3') tb as good_friend_1,good_friend_2,good_friend_3;

-- 提取Array
select dream_col
from temp_db.json_test
lateral view  json_tuple(str,'dream') dd as dreaming
lateral view explode(dreaming) tb as dream_col

--get_json_object提取指定的json元素内容(使用"$"的方式,"."表示对象,"[]"引用数组)
-- 获取标签对象
select get_json_object(str,'$.name') as name
from temp_db.json_test;

-- 获取标签中的数组元素
select get_json_object(str,'$.dream[0]') as good_friend
from temp_db.json_test;

-- 获取多层中的对象
select get_json_object(str,'$.friend.friend_1') as good_friend
from temp_db.json_test;
```

json_tuple与get_json_object都是hive自带的UDF。json_tuple 相对于 get_json_object 的优势就是一次可以解析多个 Json 字段。
       

