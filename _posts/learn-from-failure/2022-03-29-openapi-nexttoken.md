---
layout: post
title: OpenAPI设计之NextToken
tags: [learn-from-failure, java, next-token, openapi]
lang: zh
---

# 背景: 
在API设计时, 如果是`list`接口, 通常需要对返回的结果进行分页.

# 为啥需要分页?
1. 数据库层面: 防止由于没有limit, 导致从数据库中一次查询数据太多, 导致形成慢SQL: 
   1. 导致mysql需要把结果放在内存, 从而导致整体MySQL内存占用增加, 甚至oom
   2. 导致通过tcp数据库连接, 往client端传输耗时过久. 例如10MB的数据, 在100Mbps的带宽下需要几乎1秒
2. 应用层面: 如果没有分页, 也很容易造成应用内存占用过多, 如果是Java应用很可能触发FGC等.
3. 前端: 如果不分页, 导致浏览器无法缓存大量数据, 卡死.
所以一般对于非宽表, 以500~2000作为pageSize较为合适.

# 分页: pageNo+pageSize 方式

## 典型常规的分页写法

- 当页数少时, 如下第1页, 耗时0ms:

```sql
mysql> select * from employees  order by emp_no desc limit 0, 3;
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
| 499999 | 1958-05-01 | Sachin     | Tsukuda   | M      | 1997-11-30 |
| 499998 | 1956-09-05 | Patricia   | Breugel   | M      | 1993-10-13 |
| 499997 | 1961-08-03 | Berhard    | Lenart    | M      | 1986-04-21 |
+--------+------------+------------+-----------+--------+------------+
3 rows in set (0.00 sec)
```

-- 当页数多时, 如下到第300000/3=10w页, 耗时110ms: 

```sql
mysql>  select * from employees  order by emp_no desc limit 300000, 3;
+--------+------------+------------+------------+--------+------------+
| emp_no | birth_date | first_name | last_name  | gender | hire_date  |
+--------+------------+------------+------------+--------+------------+
|  10024 | 1958-09-05 | Suzette    | Pettey     | F      | 1997-05-19 |
|  10023 | 1953-09-29 | Bojan      | Montemayor | F      | 1989-12-17 |
|  10022 | 1952-07-08 | Shahaf     | Famili     | M      | 1995-08-22 |
+--------+------------+------------+------------+--------+------------+
3 rows in set (0.11 sec)
```


## 问题
随着翻页的继续, 当 `limit 1000000, 100` 的时候, 性能会急剧下降. 
是由于MySQL会把where&order之后结果集保存下来, 然后再对结果集从0~1000000进行遍历.


# 分页: nextToken+maxResults 方式

- nextToken本质上就是按照查询条件, 上一次结果集中最后一条记录的数据库主键ID值.
- 因此直接走主键索引.

## 写法

- 当第一页时, 由于没有nextToken, 因此直接获取(可以看到与pageSize写法的limit 0, 3结果是一样的): 

```sql
mysql> select * from employees order by emp_no desc limit 3;
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
| 499999 | 1958-05-01 | Sachin     | Tsukuda   | M      | 1997-11-30 |
| 499998 | 1956-09-05 | Patricia   | Breugel   | M      | 1993-10-13 |
| 499997 | 1961-08-03 | Berhard    | Lenart    | M      | 1986-04-21 |
+--------+------------+------------+-----------+--------+------------+
3 rows in set (0.01 sec)
```

- 当第10w页时, 有了上一次的nextToken, 因此如下:

```sql
mysql> select * from employees where emp_no<10025 order by emp_no desc limit 3;
+--------+------------+------------+------------+--------+------------+
| emp_no | birth_date | first_name | last_name  | gender | hire_date  |
+--------+------------+------------+------------+--------+------------+
|  10024 | 1958-09-05 | Suzette    | Pettey     | F      | 1997-05-19 |
|  10023 | 1953-09-29 | Bojan      | Montemayor | F      | 1989-12-17 |
|  10022 | 1952-07-08 | Shahaf     | Famili     | M      | 1995-08-22 |
+--------+------------+------------+------------+--------+------------+
3 rows in set (0.00 sec)
```

## 总结
可以看到, nextToken方式, 随着页数的增加, 结果与`pageSize`方式是完全等价的, 但时间消耗完全是稳定的.

# 优缺点比较与思考
nextToken方式
- 优点: 
  - 不会随着页数的增加, 导致时间消耗线性增长. 而是直接走主键索引, 时间消耗稳定.
- 缺点: 
  - 由于nextToken只有一个值, 翻页之后, nextToken就被覆盖掉了. 只能顺序向前翻页, 而很难再退回上一页. (但实际调用端可以保存下来, )
  - nextToken值本身由于是数据库主键, 因此需要进行加密, 防止外部窥探到数据条数等敏感信息. 但加解密带了了额外的CPU消耗.

# 一些NextToken+MaxResults的实践

- AWS的[DescribeInstances](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstances.html)
- AWS的[LastEvaluatedKey](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.Pagination.html), 本质与NextToken是一样的
> the LastEvaluatedKey from a Query response should be used as the ExclusiveStartKey for the next Query request

- Aliyun的[DescribeInstances](https://help.aliyun.com/document_detail/25506.html)


# 其他记录
## MySQL公共大数据集
> 为了测试大数据集nextToken与pageNo方式, 找了很多测试大数据集, 总结如下
 
- MySQL官方的[test_db](https://github.com/datacharmer/test_db)
  - 优点: 使用起来非常简单, 按照github中的步骤, 很快就完成了数据的导入.
  - 缺点: 
    - 总体数据量不太多, 最大的`salaries`表也只有200w行左右数据, 针对翻页极端情况,
    - `salaries`表没有primary_key, 不符合第二范式, 即数据表每一个实例或者行必须被唯一标识, 导致不好验证next_token
- MySQL官方的[airportdb](https://dev.mysql.com/doc/airportdb/en/airportdb-introduction.html)
  - 优点: 数据量大, `bookings`表有5000w数据, 且符合第二范式, 可以进行nextToken验证.
  - 缺点: 
    - 导入数据需要单独安装[MySQL Shell](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-install-macos-quick.html)
    - 数据集600MB, 下载速度实在是捉急.