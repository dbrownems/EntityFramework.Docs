---
title: Complex Query Operators - EF Core
author: smitpatel
ms.date: 10/03/2019
ms.assetid: 2e187a2a-4072-4198-9040-aaad68e424fd
uid: core/querying/complex-query-operators
---
# Complex Query Operators

Language Integrated Query (LINQ) contains many complex operators, which combine multiple data sources or does complex processing. Not all LINQ operators have suitable translation in the server side. Sometimes, query in one form translates to server but if written in different form doesn't translate even if the result is same. This page describes some of the complex operators and their supported variations. In future, we may recognize more pattern and add their translations. Important point to remember is that database also varies in what they support. A particular query, which is translated in SqlServer, mayn't work for SQLite database.

> [!TIP]
> You can view this article's [sample](https://github.com/aspnet/EntityFramework.Docs/tree/master/samples/core/Querying) on GitHub.

## Join

LINQ Join operator allows you to connect two data sources based on the key selector on each source and generates a tuple of values when key matches. It naturally translates to `INNER JOIN` on relational databases. While LINQ Join has outer & inner key selectors, databases take a condition as join condition. So EF Core will generate join condition by comparing outer key selector to inner key selector through equality operation. Further, if key selectors are anonymous types, EF Core will generate join condition to do equality component wise.

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#Join)]

```SQL
SELECT [p].[PersonId], [p].[Name], [p].[PhotoId], [p0].[PersonPhotoId], [p0].[Caption], [p0].[Photo]
FROM [PersonPhoto] AS [p0]
INNER JOIN [Person] AS [p] ON [p0].[PersonPhotoId] = [p].[PhotoId]
```

## GroupJoin

LINQ GroupJoin operator allows you to connect two data sources in similar to Join but it creates a group of inner values for matching outer elements. Executing query like following generates result of `Blog` & `IEnumerable<Post>`. Since databases (especially relational databases) don't support representing a collection of client-side objects, GroupJoin can't be translated to server in many cases. It requires to get all the data from server to do GroupJoin without special selector. But if the selector is limiting data being selected then fetching all data from server may cause performance issues. That's why EF Core doesn't translate GroupJoin.

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#GroupJoin)]

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#GroupJoinComposed)]

## SelectMany

LINQ SelectMany operator allows you to enumerate over collection selector for each outer element and generate tuples of value from each data source. In a way, it's a join but without any condition so every outer element is connected with element from collection source. Depending on how collection selector is related to outer data source, SelectMany can translate to various different queries on server side.

### Collection selector doesn't reference outer

When collection selector isn't referencing anything from the outer source, the result is a cartesian product of both data sources. It translates to `CROSS JOIN` in relational databases.

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#SelectManyConvertedToCrossJoin)]

```SQL
SELECT [b].[BlogId], [b].[OwnerId], [b].[Rating], [b].[Url], [p].[PostId], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Rating], [p].[Title]
FROM [Blogs] AS [b]
CROSS JOIN [Posts] AS [p]
```

### Collection selector references outer in where clause

When collection selector has a where predicate, which is referencing the outer element, then EF Core translates it to join on database and uses the predicate as join condition. Normally it arises when using collection navigation on outer source as collection selector. If the collection is empty for outer element, then it wouldn't generate any result for that outer element. But if `DefaultIfEmpty` is applied on collection selector then outer element will be connected with default value of inner element. Because of this distinction, this kind of queries translates to `INNER JOIN` in the absence of `DefaultIfEmpty` and `LEFT JOIN` when `DefaultIfEmpty` is applied.

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#SelectManyConvertedToJoin)]

```SQL
SELECT [b].[BlogId], [b].[OwnerId], [b].[Rating], [b].[Url], [p].[PostId], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Rating], [p].[Title]
FROM [Blogs] AS [b]
INNER JOIN [Posts] AS [p] ON [b].[BlogId] = [p].[BlogId]

SELECT [b].[BlogId], [b].[OwnerId], [b].[Rating], [b].[Url], [p].[PostId], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Rating], [p].[Title]
FROM [Blogs] AS [b]
LEFT JOIN [Posts] AS [p] ON [b].[BlogId] = [p].[BlogId]
```

### Collection selector references outer in non-predicate

When collection selector has reference to outer, which isn't in predicate (as the above case), it can't be converted to join. That's why we need to evaluate collection selector for each outer element. It translates to `APPLY` operation in relational databases. If the collection is empty for outer element, then it wouldn't generate any result for that outer element. But if `DefaultIfEmpty` is applied on collection selector then outer element will be connected with default value of inner element. Because of this distinction, this kind of queries translates to `CROSS APPLY` in the absence of `DefaultIfEmpty` and `OUTER APPLY` when `DefaultIfEmpty` is applied. Sqlite doesn't support `APPLY` operator.

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#SelectManyConvertedToApply)]

```SQL
SELECT [b].[BlogId], [b].[OwnerId], [b].[Rating], [b].[Url], ([b].[Url] + N'=>') + [p].[Title] AS [p]
FROM [Blogs] AS [b]
CROSS APPLY [Posts] AS [p]

SELECT [b].[BlogId], [b].[OwnerId], [b].[Rating], [b].[Url], ([b].[Url] + N'=>') + [p].[Title] AS [p]
FROM [Blogs] AS [b]
OUTER APPLY [Posts] AS [p]
```

## GroupBy

LINQ GroupBy operator creates result of type `IGrouping<TKey, TElement>` where `TKey` & `TElement` could be arbitrary types. Further, `IGrouping` implements `IEnumerable<TElement>`, which means you can compose over using any LINQ operator after grouping. Since no database structure can represent IGrouping, Group By operator has no translation in most cases. When an aggregate operator is applied to each group to which gives back a scalar, it can be translated to SQL `GROUP BY` in relational databases. SQL `GROUP BY` is restrictive too. It requires you to group by only scalar values  projection can contain only grouping key columns or aggregate applied over any column. EF Core identifies this subset and translates it to server. Following example shows a query, which has aggregate operator applied to it and generated SQL.

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#GroupBy)]

```SQL
SELECT [p].[AuthorId] AS [Key], COUNT(*) AS [Count]
FROM [Posts] AS [p]
GROUP BY [p].[AuthorId]
```

EF Core also translates when aggregate operator appears in Where filter by using `HAVING` clause in SQL and aggregate operator appears in Order By (or other ordering operators). The query before applying Group By operator can be any complex query using various different operators as long as it can be translated to server. Further, once you apply aggregate operator on grouping query to remove groupings from resulting source, you can compose it like any other query.

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#GroupByFilter)]

```SQL
SELECT [p].[AuthorId] AS [Key], COUNT(*) AS [Count]
FROM [Posts] AS [p]
GROUP BY [p].[AuthorId]
HAVING COUNT(*) > 0
ORDER BY [p].[AuthorId]
```

The aggregate operators EF Core supports are following

- Average
- Count
- LongCount
- Max
- Min
- Sum

## Left Join

While Left Join isn't a LINQ operator, relational database has concept of Left Join which is frequently used in queries. A particular pattern in LINQ query gives same result as `LEFT JOIN` on server. EF Core identifies such pattern and generate equivalent `LEFT JOIN` on server side. The pattern involves creating GroupJoin between both the data sources and then flattening out the grouping by using SelectMany operator with DefaultIfEmpty on the grouping source to match null when inner doesn't have related element. Following example shows how that pattern looks like what it generates.

[!code-csharp[Main](../../../samples/core/Querying/ComplexQuery/Sample.cs#LeftJoin)]

```SQL
SELECT [b].[BlogId], [b].[OwnerId], [b].[Rating], [b].[Url], [p].[PostId], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Rating], [p].[Title]
FROM [Blogs] AS [b]
LEFT JOIN [Posts] AS [p] ON [b].[BlogId] = [p].[BlogId]
```

Above pattern creates complex structure in expression tree. Because of that EF Core requires you to flatten out grouping results out of GroupJoin operator in immediate step. Even if the GroupJoin-DefaultIfEmpty-SelectMany is used but in different pattern, we may not identify as Left Join.
