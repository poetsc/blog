---
title: join vs sub-query
date: 2020-04-09 23:27:01
tags: [mysql]
---
>A LEFT [OUTER] JOIN can be faster than an equivalent subquery because the server might be able to optimize it better—a fact that is not specific to MySQL Server alone.

参考：[Rewriting Subqueries as Joins](https://dev.mysql.com/doc/refman/5.7/en/rewriting-subqueries.html)