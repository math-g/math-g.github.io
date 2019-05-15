---
layout: post
title:  "Selective activation of integration tests with Gradle, without specific sourceSet"
comments: true
date:   2019-05-15 23:21:58 +0200
---

Integration tests typically need more resources setup, and having them enabled all the time is not convenient, as it slows down the builds and the development velocity.
Sometimes also, needed resources are not available on a particular development environment and the build setup cannot be done.
So we want to be able to run the build with unit tests, but without integration tests, to run integration tests only, to run both etc.
This is a classical problem.

And the classical solution with Gradle is to use a specific `sourceSet` for integration tests, and a custom `integrationTests` task (of type `Test`), with the `testClassesDirs` property set to this `sourceSet` output classes.

Using dependancies declaration (as `check.dependsOn integrationTest` and `integrationTest.mustRunAfter test`), we can force the build task to run the integration tests after the unit tests. We can also run the `integrationTests` task independently.

So the classical approach relies on Gradle's task definition capabilities, and custom commands, to actually do the selective activation of integration tests.

Another possibility I recently developed, is rather to use the selective activation capability of the unit test framework, and define the conditional logic with Groovy.
Quite curiously, I did not find any example of it (even my "Gradle In Action" book use the sourceSet method).

I tend to prefer this other approach, as I find it more straightforward to implement and easy to maintain.
Particularly, there is no custom sourceSet : it is not the build script which knows which tests are unit tests or integration tests, but the tests themselves, modifying an annotation.
This way, if needed, a unit test can easily become an integration test and vice versa.

Finally, I found it more powerful, allowing to activate selectively subsets of integration tests, from a simple groovy data stucture, as a map.

For instance, I can declare the following map in `settings.gradle` :
    
	gradle.ext.integrationTests = ['POSTGRESQL':true, 'ORACLE':false]
	
The line declares that I want to enable PostgreSql related integration tests, but disable the Oracle ones. 

The goal is to be able to provide an environment variable to the tests framework, to selectively activate all the corresponding (PostgreSql related) integration tests classes.
Using Spock, this could be done like this :

    import spock.lang.IgnoreIf
    import spock.lang.Specification
	
    @IgnoreIf({!(env.INTEGRATION_TEST_POSTGRES == 'true')})
    class PostgreSqlSpec extends Specification {
	   [...]
	}
	
Yes that's a kind of double negation logic. I just tend to use only @IgnoreIf in one project for test classes, whatever the condition is. But @Requires could also be used.

The needed environment variable is declared like this in `build.gradle`, with its value retrieved from `settings.gradle` :

    test {
        [...]
        environment "INTEGRATION_TEST_POSTGRES", gradle.integrationTests['POSTGRESQL']
        [...]
	}

The user will only modify the `settings.gradle` file (because build.gradle is not a config file and should not be updated by operators).
	
We also typically need some specific setup for the integration tests, like files, connections, containers etc.
But we don't want to pollute the main `build.gradle` file with those details.
So a specific gradle script can be executed like this :

    if (gradle.integrationTests['POSTGRESQL'] == true) {
        apply from: 'src/test/groovy/my/package/integrationTest/postgresql.gradle'
        afterEvaluate {
            test.dependsOn pgSetupTask
            test.finalizedBy pgResourcesShutdownTask
        }
    }
	
An `afterEvaluate` method is used to link the execution of the integration tests gradle script with the main Test task.
`pgStartupTask` and `pgResourcesCleanupTask` are example task names for integration tests setup and cleanout (code not shown), defined in `postgresql.gradle`.
