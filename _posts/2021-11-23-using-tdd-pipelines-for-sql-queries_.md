---
layout: post
title:  "Test Driven Development of complex SQL queries with Gitlab pipelines and Spock"
comments: true
date:   2021-11-23 23:21:58 +0200
---

SQL is a central skill in data engineering / business intelligence projects, and, as SQL is a language, SQL developpers are developpers. So we should be able to use classical development tools and methodologies that we see for mainstream programming languages, as unit testing and test driven development, applied to SQL.  
But that is very often not the case, and modern testing in data engineering seem sometimes to be still an open problem.  

Here I will show you a work I developed last year to provide pipelined testing to a SQL repository, which has been successfully used by other teams members, and now generalized as a development method, in which I am not required anymore.

### Testing datasets is hard

So why in the first place TDD is so hard to setup for SQL or data pipelines development ?

First reason is the nature of SQL output, that is tables (or resultsets, datasets as you prefer), that can be large. Testing tables means checking its entire content. It is not as easy as checking the state of a class or the output of a function after a specific call. Most often, incorrect SQL still executes and don't produce any error to catch. What we have to detect instead is an incorrect content : with a table, if we have 1 million rows containing only one incorrect row, the test must fail. But how can we be sure that the expected 1 million rows to test against are correct ? That is equivalent of the definition of the correct assertions validating a class or a system in classic TDD. The assertions are an automatically testable specification and for SQL, this specification exist also : it is the business question this query must answer to.  
So the test must concentrate the essence of the business question this SQL was writed for, without any additional data (ie additional columns), and that is a full-fledged business intelligence work task, that can be quite time consuming, as we will describe here. Maybe that is one reason explaining why in my experience and what I could hear from other places, there are few SQL tests for SQL codes.
Moreover, to be assertable, the dataset to test against must come from the source of truth. It means it has to come the most directly possible from the source systems, and share ideally no transformation logic with the tests code, so that errors in data transformations would not propagate to it.

Another reason can be the skillset of the engineers affected to BI jobs, that has been seen traditionnally as a separate population than software developers. This was probably caused by the classic BI tools (ETL, dashboards) that were GUI oriented, and for quite a long time, BI jobs had been assimilated to parameterization of these interfaces and have lived many years in a distinct world, surprisjngly exempted from solid testing obligations.  
Tests are done of course, but mostly at initial development and deployments times, and then only challenged by dashboard users over time, asking to make a correction for a some case they found incorrect when using the report. Other errors, for instance a very rare occurence missing in the dataset, or a new use case from which reports users would not be aware, would not be systematically catched.

Data engineering is a practice certainly trying in itw own denomination to differentiate itself from the old school BI ones. So there is a focus on applying software engineering tools and methods to dataset / data pipelines development, but I can still observe that a lot of focus is made on tools usage rather than business exactitude.

### How to test SQL queries
As we said, the output of a SQL query is a table, so to test the correctness of a table using the same strategy than for testing classes or functions, we would test attributes of the table. We could test the number of rows, the presence of the right IDs etc.
It is indeed an approach that I have seen.
But the problem is that source data is always changing. So we would have to test against a stable environment, which is possible if restricting our tests to a read only development database.
The dataset to test against - let us call it reference dataset - would be exported from a production or integration environment, after having being successfully validated by business users.
When adding new features to the query, we have to determine if this reference dataset is invalidated or no. For instance, simply adding a new column already present in one of the queried tables, without any criteria, won't change the number of rows and the expected ID list.
In this case, we can continue to test the query output after the addition of the column.
On the other hand, if the query output is invalidated, because for instance a criteria was added in the WHERE clause, it is not possible to use the reference dataset anymore. We have to start again, validating a query output with business users on a read only environment, exporting it and using it in the subsequent iterations.  
This is not convenient nor agile because this means there is an iteration starting with the new requirement where we don't have a valid reference dataset anymore to check our developments.  
We also have the same problem if want to test (even develop) in a non-read-only environment : in this case, the data is continuously changing and the idea of a reference dataset can't stand anymore. 
So what is needed is not a reference dataset, but a reference query.  
This would be a query providing in its output the correct number of rows and ID list, instead of the output itself.  But wait, this query is the query I am tasked to develop no ?
Are we running in circles ?   
Actually no. Because of two reasons :
 - With SQL, there is alway many ways to do the same task.
 - A lot of error prone elements in a query, as complex joins or subqueries to retrieve a column, are not expecting to change the expected number of rows and ID list.

So to create a good reference query, we must use another way to answer the same business question, and keeping only the strictly essential elements needed to produce the same number of rows and same list of IDs (we call this combination the "perimeter").

If developed query and reference query converge, we gain a lot of confidence in the correctness of our development.  
With this approach of developing reference queries, we find again the same classic burden of double maintenance.  
But we have now an asset providing the great benefit of being able to test at all times, independantly of the non agile verification by business users.

Armed with these reference queries, we can now setup a modern test pipeline.  
I will describe next the specific constraints that led to its design.

### Using Spock to give the power of automated testing to SQL-only people.

 - Utiliser les outils devops en place : Gitlab CI
 
An essential feature that was needed in my case was to provide a test framework for developers with the most minimal need in code other than SQL.  
The reason was that some members of the team expected to work on SQL development are not data engineers. They are not fluent in mainstream test frameworks as JUnit or PyTest. They can rather be functional profiles fluent in SQL and they do not know well any language other than SQL.  
Actually, this requirement is beneficial because it leads to a clear separation of concerns, exposing to the test framework users only a few hooks related to the test SQL query, and the SQL query to be tested. 

I based my implementation on Spock, which is a test framework based on JUnit runner, using the power of Groovy compile-time metaprogramming to expose test specifications looking like natural language.

I will classicaly call the teamp members developing SQL, the developers.

With Spock, test classes are called Spec and must inherit the Specification class.
To add unit tests for a SQL query, developers repeat the same process : 

 - Creation of a class inheriting a parent Specification containing custom method to create fixtures et achieve the comparison. Although it is a Groovy class, they actually copy and paste a template and they can ignore the concept of class and methods.
 
 - Override of methods providing the tested query and the reference query : it is basically relative references to .sql files in the repo.
  
 - implementation of Spock test methods.
  
 - add lines in a common file mapping the Specification class to the tested and reference queries. This file will be used to decide which specific Specification to run after a commit.
  
Simplified exemple with dummy Specifications names:

    @Stepwise
    class MySpec extends CommonSpec {
    
        @Override
        String getTargetTableName() {
            "my_specific_report"
        }
    
        @Override
        protected String getTestQuery() {
            """
                SELECT 'test_query' AS query_type, object_id, ([scalar query returning site]) AS site
                FROM (
                    ${getQueryString('relative/path/to/query/file/query_to_test.sql")}
                ) detail
            """
        }
        
        @Override
        protected String getReferenceQuery() {
            """
                SELECT 'ref_query' AS query_type, object_id, ([scalar query returning site]) AS site
                FROM (
                    ${getQueryString('relative/path/to/query/file/reference_query.sql")}
                ) ref
            """
        }
        
        def """The number of object_id for #site is correct"""(){
            given:
            def testQuery = "SELECT object_id FROM $targetTableName WHERE type_req = 'test_query' AND site = '$site'"
            def refQuery = "SELECT object_id FROM $targetTableName WHERE type_req = 'ref_query' AND site = '$site'"
        
            when:
            def comparison = compare(reqTest, reqRef)
        
            then:
            comparison.equality
        
            where:
            site << sites
        }
        
        [other tests]
        ...
    
    }
    
Test class to be developed inherits from the abstract class `CommonSpec` (name changed) which in turn, inherits from Spock's `Specification` class.
`CommonSpec` is a quite sophisticated class providing services as connections handling, setup of ephemeral target database to load test data in a postgres container, parallel execution of queries over multiple sites, comparison of datasets and so on.
Developers are not expected to understand it, but just to make their test classes inheriting from it.
Once it is done, Sseveral methods can be overrided to provide customizations (for instance, to specify an `ALTER SESSION` command to be executed before the tests).
A least two methods must be overrided and where shown in the code above :
 - A method to provide the query to be tested.
 - A method to provide the reference query.

Actually, there are two possible modes :
 - A mode where both queries are provided separately, as shown here. In this cas, the `CommonSpec` class is able to execute both queries in parallel, started in the same method.
 So the source database where both queries are executed will see two distinct sessions, started at the same time, with a start timestamp differing only by a few milliseconds at the very most.
 - A mode where only one query is provided combining both tested and reference query combined in a same custom SQL, typically with a `NATURAL FULL OUTER JOIN` (as described [here]).
 This mode is useful for test on real time data, where even a few milliseconds between start timestamps of distinct sessions as in the previous mode could produce deltas.

The logic behing the compare method is inherited from the common parent Specification.
What it does is the following :

That is all they have to do, and after more than one year working with this approach, now we can safely say that even the most non technical developers are able to follow it.  

When this setup is in place, developer can work on the SQL thay have to implement (based on the current user story), and on commit, a test suite containing their Specification is automatically run. Results are then available on a dedicated web page.  

This automatic process is now described.

IN PROGRESS

### Integration with local CI tools

[I had also to integrate with existing CI tools, that had beed generalized in the company]

IN PROGRESS

## Conclusion

 - [Team productivity can then be increased tenfold as well as confidence before deployments in production, and error rates  are minimized]..
 
 IN PROGRESS
 
