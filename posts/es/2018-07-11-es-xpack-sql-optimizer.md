---
title: "x-pack sql做的sql优化"
date: "2018-07-11T06:33:33.000Z"
tags:
  - "es"
source_url: "https://gpdream.github.io/2018/07/11/es/es-xpack-sql-optimizer/"
recovery_source: "html"
---
# x-pack sql做的sql优化

es 6.3的插件xpack内置了sql, [和NPLChina的elasticsearch-sql](https://github.com/NLPchina/elasticsearch-sql)插件相比,要少很多实用性的功能,但相比有蛮多性能上的优势。今天看看都有哪些sql优化点。

## x-pack sql做的sql优化

- **PruneDuplicatesInGroupBy:** 裁剪重复的group by 字段。以sql`select * from table group by code,code`,这里两次group by都是code,在[elasticsearch-sql](https://github.com/NLPchina/elasticsearch-sql)中,会被group by两次重复计算。xpack-sql做了这个优化，然而是一个没用的优化，因为xpack-sql不支持多字段group by，直接报错。
- **ReplaceDuplicateAggsWithReferences:** 对聚合函数的别名和函数做统一列名。以sql `select count(*) c,count(*) from index`为例,`count(*) c` 和`count(*)` 都会被统一成`count(*) c`来处理掉。性能上并不会因为这个有什么变化。
- **ReplaceAggsWithMatrixStats:** 把`SKEWNESS,KURTOSIS`使用`matrix_stats`来替代掉，matrix_stats是es提前统计好的，这么做会有很大的性能提升，不需要真的去计算斜率和偏度系数。
- **ReplaceAggsWithExtendedStats :** 对`SUM_OF_SQUARES, STDDEV_POP, VAR_POP`等方法转换成使用es提前计算好的`extended_stats`的方法。
- **ReplaceAggsWithStats:** 对`min,max,avg,sum`方法直接调用es的`stats`方法，而不是真的去查询遍历，这个对性能有非常大的提升。跟前面两个的性质一样。
- **PromoteStatsToExtendedStats,ReplaceAggsWithPercentiles,ReplceAggsWithPercentileRanks:** 对`PERCENTILE和PERCENTILE_RANK`方法使用。
- **CombineProjections:**  合并掉一些相同的别名之类的东西。
- **ReplaceFoldableAttributes:** 替换掉已知的属性，举例`ELECT 5 a, 3 + 2 b ... WHERE a < 10 ORDER BY b`,那么a和b不需要去数据库中查询,直接给替换掉。
- **ConstantFolding:**  针对别名做的优化，多个别名重复之类的。
- **BooleanSimplification:**  一些简单的boolean计算直接计算出来，不走数据库，比如 2=2 and 1=1之类的。
- **BinaryComparisonSimplification:** 一些简单的>=,<之类的二进制大小比较。
- **BooleanLiteralsOnTheRight:** 从源码上看如果是BinaryExpression,并且右边表达式是Literal，则会替换掉左右的值。比如field = 10，会被替换成10 = field.
- **CombineComparisonsIntoRange :** 合并掉>/>= AND </<= ,</<= AND >/>=的比较区间。 比如 a>5 and a>10 给合并成了a>10 。
- **PruneFilters:** 裁剪filter,只对Literal的condition生效,Literal的Condition能直接生效立即知道true还是false.
- **PruneOrderBy:** 裁剪Order by,1. 移除order by的常量，例如 `select 1 a,field from table order by a,field` ,那么a会被裁剪掉
- **PruneOrderByNestedFields:**
- **PruneCast:**
- **SkipQueryOnLimitZero:** 如果是limit 0则直接跳过查询
- **SkipQueryIfFoldingProjection:** 如果是已经找到值的则跳过查询

做了蛮多sql语句层的优化,但是并没有找到类似CBO那样的优化，不过es自己在底层做查询的时候本身有针对倒排索引的优化，就算xpack-sql里没有也不是太大的问题。
