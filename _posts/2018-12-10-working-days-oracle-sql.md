---
layout: post
title:  "Working days duration calculation in Oracle SQL, without any DDL"
comments: true
date:   2018-12-10 23:38:58 +0200
---

Hi, this is my first post here. I will try to expose some of my work, mainly about data engineering.

Here I start with some Groovy code I had to write to deal with a classical problem for which I paradoxically did not find a lot of good solutions : the working days duration calculation (ie how many business days there are between two dates), all in SQL.
My situation was actually even more restrictive because I could not create any object in the schema, in other words, no DDL were allowed.
This calculation is needed for a lot of datasources, used for many reports, and exact results are needed, as critical business rules can be fired or not depending on expected durations.
Moreover, I work in France (Paris) and french working days calendar is not the same from year to year because of some national holidays that are dependant of the Easter sunday date, which change every year. The precise date is given by a quite sophisticated, but deterministic, calculation, making use of lunar cycles and spring equinox (by the way I realised that is probably the most scientific part of religion).
The database is Oracle and some PL/SQL packages exist, most of the time specific to some project. But again in my case, using DDL was not an option, because I could not update the schema in any way. This is because query had to be executed on a schema owned by a software editor.
So I thought it was worth taking the time to do this calculation properly.
The approach is given in this repo :

[working-days-oracle-sql](https://github.com/math-g/working-days-oracle-sql)

Comments in the project give most of the details.
Project actually use PL/SQL, but as a function declared in the WITH clause. This needs Oracle 12c+. It is not mandatory and could be rewritten without a function, to port it to other database. Because no DDL were allowed, I could not use proper types to handle the dynamic holidays list returned by this function. So I used text functions (LIKE operator), that is less stylish but it worked.
Durations are always relative to current date. Past durations differ from future one because in both cases, only the start date is inclusive, and this start date is not the same (for past duratins, it is the start date whereas for future ones it the the current date). It certainly need some refactoring to harmonize the way both calculations are done, but for now tests all pass (I will update the project later).

You can see one of my favorite way to work with SQL generation, that is working with Groovy and GStrings (more on that later).
I always use TDD like approach even for analytics jobs, and you can see how I used unit testing to validate this work.
This is done with Spock. Here I make extensive use of the @Unroll annotation to parameterize the tests, even using a "testCase" variable to label each unrolled test.
