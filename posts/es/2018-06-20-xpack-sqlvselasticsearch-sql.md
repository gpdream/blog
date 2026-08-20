---
title: "es 6.3内置sql vs NLPChina elasticsearch-sql"
date: "2018-06-20T08:33:33.000Z"
tags:
  - "es"
source_url: "https://gpdream.github.io/2018/06/20/es/xpack-sqlvselasticsearch-sql/"
recovery_source: "html"
---
# es 6.3内置sql vs NLPChina elasticsearch-sql

es 6.3发布了,内置sql插件xpack-sql。在es内置sql插件之前，一直都是使用的NLPChina的elasicsearch-sql插件。本文重点对比xpack-sql和elasticsearch-sql的差异。从功能，函数，sql兼容度和性能上对比。\

## 功能对比

| 功能 | xpack-sql | elasitcsearch-sql | 描述 |
| --- | --- | --- | --- |
| = | 支持 | 支持 |  |
| 不等于 | &lt;&gt;,!=,&lt;=&gt; | &lt;&gt; | 有一个就够了 |
| 比较大小 | &lt;, &lt;=, &gt;, &gt;= | &lt;, &lt;=, &gt;, &gt;= | 相同 |
| between and | 支持 | 支持 |  |
| 是否为空 | is null/is null null | is missing,is not missing | 写法不同 |
| 逻辑操作 | and / or / not | and or | es-sql少个not，不过无所谓 |
| 数学操作 | + - * / % | 要借助script脚本来支持 | es-sql的写起来明显要更麻烦 |
| 内置数学函数 | abs,sin,log等等 | flow,split,trim,log等等 | 数学函数较少,但是有一些其他更实用的函数 ,后面给详细函数对比 |
| 时间函数 | MINUTE_OF_HOUR等多种时间函数 | 无 | 后面给详细函数对比 |
| 聚合函数 | avg,count,min,max,sum | 相同 |  |
| order by | 支持 | 支持 | 相同 |
| 深度分页 | 支持cursor来达到滚动的方式，但并不是真的特别好用 | 用hint语法USE_SCROLL 来达成 | 一个比一个更不好用系列 |
| include(fields),exclude(fields) | 不支持 | 支持 | 比较鸡肋的一个功能，可以少写几个字段名 |
| IN_TERMS | 不支持,得写sql+dsl | 支持 |  |
| TERM | 不支持，但是会自己解析成term | 支持 |  |
| IDS_QUERY | 不支持 | id in还是需要的 |  |
| stats(field) | 不支持 | 返回min,max,count,sum,avg等结果 |  |
| es querystring，类似 q=query(‘address:880 Holmes Lane’) | 不支持，有类似match的说法 | 支持 | 比较有用的功能，勉强算都有吧 |
| es script | 不支持，但是可以直接写dsl和sql一起用 | 支持 | 算是都支持把，但写起来都别扭。尤其xpack-sql的dsl和sql混写，更加别扭 |
| geo查询 | 暂未找到，应该有吧？ | 支持 | xpack-sql的没找到，不知道是有还是没有 |
| 高亮 | 无 | 支持 | 有用的功能 |
| hits | 无 | 支持 | 很强大很神奇很反人类并排不上用场的功能 |
| join | 不支持 | 支持的很鸡肋 | es-sql的这个支持等于不支持 |
| nested | 没找到 | 支持 | 在Json字段查询时候，很有用的一个功能. |
| union/minus | 不支持 | 支持 |  |

功能上,NPLChina的elasticsearch-sql略胜一筹。

## 函数对比

| 函数名 | xpack-sql | elasticsearch-sql | 备注 |
| --- | --- | --- | --- |
| ABS绝对值 | 支持 | 不支持 |  |
| CBRT立方根 | 支持 | 不支持 |  |
| CEIL/CEILING向上舍入 | 支持 | 不支持 |  |
| E自然底数,返回2.7182818284590452354 | 支持 | 不支持 | 该函数返回 |
| round，四舍五入 | 支持 | 支持 |  |
| log | 支持 | 支持 |  |
| log10 | 支持 | 支持 |  |
| sqrt平方根 | 支持 | 支持 |  |
| exp,e的x次方 | 支持 | 不支持 |  |
| 一大堆三角函数 | 支持 | 不支持 | sin,cos等等十来个 |
| 时间函数 | 取年、月、日、分秒 | date_format函数 | 这时间函数有点尴尬啊 |
| substring | 不支持 | 支持 | 很重要的函数 |
| concat_ws拼接字符串 | 不支持 | 支持 | 很常用的函数 |
| split | 不支持 | 支持 | split也是常用的 |
| range，group by时候用 | 不支持 | 支持 | 挺方便的功能 |
| date_range,作用于时间的range | 不支持，支持 | 同上 |  |
| date_histogram，group by的时候用 | 不支持 | 支持 |  |

综上,xpack-sql比elasticsearch-sql多非常多的数学函数以及时间操作函数，但是少了字符串函数和用于group by的函数.

## sql兼容性

sql兼容性对比是个很尴尬的事情,这两都不是标准sql,不能以sql标准测试集来测。两人的支持又存在较大的差异。

| sql | xpack-sql | elasticsearch-sql | 备注 |
| --- | --- | --- | --- |
| 多索引查询,select * from index1,index2 | 不支持 | 支持 | 看到这个不支持，心凉了一半 |
| 多字段group by,select count(*) from index 1 group by f1,f2 | 不支持 | 支持 | 心的另一半也凉了 |
| xpack上面两个不支持其他懒得看了 |  |  |  |

## 性能

x-pack sql比elasticsearch-sql有明显的性能优势。x-pack sql是把sql直接发到节点下去执行。且会做一些简单的性能优化。举例有个字段如下。那查询author=’a’的时候，会自动推断成author.keyword=’a’。有些类似这样的小优化在这里。

```plain
author: {
	type: "text",
	fields: {
		keyword: {
			type: "keyword",
			ignore_above: 256
		}
	}
}
```

而elasticsearch的工作方式是,把sql转换成elasticsearch dsl,再提交给elasticsearch去运行。其生成的dsl在很多场景下都比较奇葩，更不要说性能优化了。而且像uinon,minuxs之类的语法会消耗大量的io资源，不过xpack-sql都没有这个功能。
