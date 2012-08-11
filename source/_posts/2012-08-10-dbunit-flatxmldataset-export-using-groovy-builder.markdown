---
author: Doug Ly
layout: post
title: "DBUnit FlatXmlDataSet Export Using Groovy Builder"
date: 2009-03-13 20:35
comments: true
categories: programming
---

Recently I have been using [DBUnit][1] to run my DAOs unit test with Spring and HSQL in memory database. One of the biggest challenge in testing the DAO layer is generating data to test with.
Of course you could export data in your development database in style of insert statement files and import them into the test database before executing the test. But I think doing so still requires a lot of manual work
for a busy developer. One nice thing about DbUnit is that you can export your test data into a single XML file and while setting up the unit test, you tell DbUnit to execute the import based on the XML data file.
<!-- more -->

Just for reference, a FlatXmlDataSet looks like this
``` xml FlatXmlDataSet
<dataset>
    <table1 id='1' name='hello'/>
    <table1 id='2' name='world'/>

     <table2 id='1' name='hello3232'/>
    <table2 id='3' name='hello323'/>
</dataset>
```

DbUnit will create 2 rows for table1 and 2 rows for table2 in your schema. For more info please see this
Now Groovy Builder, especially the MarkupBuilder is perfect for this kind of task: extract data from a query and build the XML file based on the extracted data.
The code looks like this:

``` groovy
import groovy.sql.Sql
import groovy.xml.MarkupBuilder

sqlDriver = Sql.newInstance("jdbc:oracle:thin:@oracle_home:12345:sid", 
                            "username", 
                            "password", 
                            "oracle.jdbc.driver.OracleDriver")
queries['USER'] =
"""
select
*
from
USER
where
user_id = '12345'
"""

queries['COVERAGE'] =
"""
select
*
from
COVERAGE coo
where
coo.user_id = '12345'
"""

buffer = new StringWriter()
builder = new MarkupBuilder(buffer)

builder.dataset()
{
    queries.each
    {
        table,stmt ->
            sqlDriver.eachRow(stmt)
                    {row ->
                        map = [:]
                        (0..<row.getMetaData().columnCount).collect {
                             col -> map[row.getMetaData().getColumnName(col+1)] = row[col]
                        }
                        "${table}" (map)
                    }
    }
}

def fos = new File('dbunit_test.xml').withWriter
{
    it.write(buffer.toString())
}
```
For each query the map 'queries' define, MarkupBuilder will generate the FlatXmlDataSet format entries for the returned tuples.
There we go your test data file!
