---
author: Doug Ly
layout: post
title: "Query Templating for Hibernate Native Query"
date: 2009-07-02 14:09
comments: true
categories: programming
---

I recently encounter this kind of problematic requirement: making a native SQL in hibernate dynamic. The query should be written as a template. And based on the user selection, a concrete SQl statement is derived.
It's pointless to have this conversation if we use HQL's query criteria of course. But with native SQL code, I admit there is no easy/legitimate way around but hack.

<!-- more -->

Technology stack: Hibernate with JPA through Srping JpaTemplate helper.

So for example, suppose this is what my SQL template looks like

``` xml Hibernate Native SQL
<sql-query name="findProductCoverageSummaryForSelectedUserInTeam" resultset-ref="productCoverageResult">
  select
  product_id_level_${level},
  product_name_level_${level}
  from product
</sql-query>
```

This is what the coressponding Java code looks like

``` java Using the Native SQL
Session session = (Session) getTransactionalEntityManager().getDelegate();
Query query = session.getNamedQuery("findProductCoverageSummaryForSelectedUserInTeam");
String sql = StringUtils.replace(query.getQueryString(), "${level}", Integer.toString(productLevel));
Query q = session.createSQLQuery(sql).setResultSetMapping("productCoverageResult");
List<Object[]> results = q.list();
```

I use the native query just as a placeholder for my sql template. Then I use Apache StringUtils to replace those variables with the actual values.
I can't think of another better way to do it!

If you need a more complicated template, ones with logical condition then use Velocity as the templating engine. Velocity is what I am using but FreeMarker also is a good
candidate.

``` xml Native SQL with Velocity template
<sql-query name="findProduct" resultset-ref="productCoverageResult">
  select
  product_id_level_$level,
  product_name_level_$level
  from
  product   #if ($orderBy)   order by   $orderBy $sortType   #end
</sql-query>
```

``` java Expand Native SQL Using Velocity Template
VelocityContext ctx = new VelocityContext();
ctx.put("level", level);
ctx.put("orderBy", level);
ctx.put("sortType", sortType);
Session session = (Session) getTransactionalEntityManager().getDelegate();

Query query = session.getNamedQuery("findProduct");
String sql = null;
try
{
    StringWriter writer = new StringWriter();
    Velocity.init();
    Velocity.evaluate(ctx, writer, "VEL", query.getQueryString());
    sql = writer.getBuffer().toString();
}
catch (Exception e)
{
    throw new RuntimeException(e);
}         
       
Query q = session.createSQLQuery(sql).setResultSetMapping("productCoverageResult");
List<Object[]> results = q.list();
```
