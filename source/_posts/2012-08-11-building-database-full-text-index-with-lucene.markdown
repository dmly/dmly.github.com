---
author: Doug Ly
layout: post
title: "Building Database Full Text Index with Lucene"
date: 2010-09-14 14:15
comments: true
categories: programming
---

I have this client table in Oracle having more than 3 million rows. One of the column is the client name in which our users want to have full text search support.
Originally, Oracle context search feature was choosen by a group of consultants. May be we were maintaining the domain index correctly but its performance sucks.
<!-- more -->

I spend most of my time troubleshooting this issue and found out that we need to run "optimize" to make the index perform well. But anyway I am tired of having to hear users' complaints
everyday so I got rid of Oracle full text search and refactor the search service to use Apache Lucene.

Here is how is done:

1. Connect to the client table and get the rows need to be indexed. Of course I did this in batch of 20K rows each.
You could gain significant performance of this data loading using a database connection pool and pulling data in parallel.
2. Use Lucene to index the client name and store the client IDs for DB retrieval later on

``` groovy Extracting Data for Indexing
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import groovy.sql.Sql;
import com.mchange.v2.c3p0.ComboPooledDataSource;
import java.util.concurrent.*

ComboPooledDataSource ds = new ComboPooledDataSource()
ds.driverClass = 'oracle.jdbc.driver.OracleDriver'
ds.jdbcUrl = ''
ds.user = ''
ds.password = ''
ds.minPoolSize = 2
ds.maxPoolSize = 4
ds.acquireIncrement = 1

def stepCount = 20000
BlockingQueue queue = new ArrayBlockingQueue(8);
def pool = Executors.newFixedThreadPool(3);
def ecs = new ExecutorCompletionService(pool);
def submit = { c -> ecs.submit(c as Callable) }

def sql = """
select
id, name
from
(
select /*+ FIRST_ROW(${stepCount}) */
id, name, rownum rnum
from
atlas_client
where
id > 0
and rownum <= ?
order by
id asc
)
where
rnum > ?
"""

def maxCount = new Sql(ds).firstRow('select count(1) as maxCount from atlas_client where id > 0'').maxCount;
println "Max count: ${maxCount}"
def range = (0..maxCount).step(stepCount)
if (range[-1] < maxCount) range << maxCount

def indexDir = new File('atlas_client_index');
IndexWriter writer = new IndexWriter(indexDir, new StandardAnalyzer(), true);
writer.setUseCompoundFile(false);

def query = {low, hi ->
  println "Querying from ${low} to ${hi}"
  def rows = []
  new Sql(ds).eachRow(sql, [hi, low]) {
    row -> rows << row.toRowResult()
  }
  println "Finishing query. Return ${rows.size()} rows from ${low} to ${hi}"
  return rows;
}
 
def createIndex = {array ->
  array.each {
    Document doc = new Document();
    doc.add(new Field("id", it.id.toString(), Field.Store.YES, Field.Index.NO));
    doc.add(new Field("name", it.name, Field.Store.YES, Field.Index.ANALYZED));
    writer.addDocument(doc);
  }
}

for (int i = 0; i < range.size() - 1; i++)
{
  println "Submitting query for clients record from ${range[i]} to ${range[i+1]}"
  def low = range[i]
  def hi = range[i+1]
  submit { query(low, hi) };
}

for (int i = 0; i < range.size() - 1; i++)
{
  def results = ecs.take().get();
  println "Results (${results.size()}) ready for clients record query, indexing..."
  createIndex(results);
  println "Indexing done."
  writer.commit();
}

writer.close();
pool.shutdown();
```

