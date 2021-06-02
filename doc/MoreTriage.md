## Triaging

### Step 1. Find failures

- Find a build in TRSS (https://trss.adoptopenjdk.net/)
  - There are two sets of timed pipelines
    - 'Nightly' builds - run on an not-quite-nightly schedule
    - 'Weekly' builds - run over a weekend.  More tests are run on these builds (the 'extended' test suites are added) - so expect more failures.
    - The jdk build part of the pipeline is the same in both types of build
- Use the various TRSS views and / or Jenkins job logs to locate failing tests
  - The TRSS 'Test suite analysis' view (click the coloured block for the job in the 'Grid' view) has some 'Action' icons.  Use 'Deep history' and 'All platforms' to help see whether the failure is restricted just certain machines or platforms. There is also an Action icon to take you to the Jenkins job which ran the test.
- For openjdk tests, there are many tests within a test target. If the test target has failed you need to look for "TEST: " in the job log to find the failing test(s).
- Some tests jobs are run with the 'Parallel jobs' feature enabled, where the test suite is dynamically split into multiple jobs to minimise overall execution time.
  - In this case the apparent test job is actually a job which launches the parallel sub-jobs - and these are the jobs which contain the test output (for the tests allocated to that sub-job.
  - The TRSS 'Test suite analysis' view collates the results for all the sub-jobs (so the view is still a view for the whole test suite), and the 'Jenkins link' for the individual test targets is a link to the correct sub-job for that target.
- Do a quick review of all the test failures for a build - you are looking for an overall appreciation of what is failing
  - Are the same tests failing everywhere (may indicate an infrastructure issue, may indicate a cross platform regression, may indicate a new / changed test which doesn't work, or doesn't work in our environment) or are they unique to a platform / release (indicates machine / platform specific setup issues, platform / release specific test case issues or actual jdk issues)
- When you've honed in (what looks like) a single problem.....

### Step 2 - Raise an issue

Might the failure be due to infrastructure?
- Look for messages from the tests about expected resources not being available (e.g. expected core dump not found)
- Look for suspect operating system messages ("No space on device")

Establish where / when the test passes and fails
- Has the test ever passed (on this machine, platform, release)?
- Does the test fail everywhere - or just this machine, platform or release
- Does the test fail if you rerun on the same or a different machine?
- Does the test also fail at ci.eclipse.org/openj9?
- Does the test fail on both hotspot and openj9?  If it only fails on one then it may be a jvm issue. (Or it may be a test case issue!)
  - Not all tests run on both hotspot and openj9.
  - In particular, many more 'functional' tests are excluded on hotspot than openj9 (they are the tests written by openj9 to test openj9), and many more 'openjdk' tests are excluded on openj9 that hotspot (they are the tests written by openjdk to test openjdk).  The excluded tests are not valid on the alternative implementation.
  - So if a test fails on openj9 or hotspot but the same error cannot be seen on the other implementation it may simply be that the test is not run on the other implementation (check the exclusion list files).

Is the test new?
- Useful to know - it may never have been run in the failing environment, or it might just not work.

Is the failure aleady known? Look in
- openjdk bugs database: https://bugs.openjdk.java.net/projects/JDK/issues
- eclipse-openj9: https://github.com/eclipse-openj9/openj9/issues
- https://github.com/adoptium/aqa-tests/issues
- https://github.com/adoptium/infrastructure/issues
- https://github.com/adoptium/temurin-build/issues
- https://github.com/adoptium/aqa-systemtest (for systemtest failures)
- https://github.com/adoptium/stf (for systemtest failures)

If it's not already known, raise an issue:
- If it's obviously a machine which needs fixing, raise in https://github.com/adoptium/infrastructure/issues
- If it's obviously an eclipse-openj9 issue (doesn't fail on hotspot) - raise in https://github.com/eclipse-openj9/openj9/issues
- If it's a systemtest issue, raise in https://github.com/adoptium/aqa-systemtest
- If it's not one of the above,  raise in https://github.com/adoptium/aqa-tests

If it is already known:
- Are you able to add any more information to the existing issue which might help in debugging the failure?
- It is useful just to add "also seen on platform / release xxx" if that information is missing
- It is also useful to add "still occurring" so that people reviewing old issues know it still needs attention

For functional or system test failures where the failure is unique to openj9 (the same failure does not occur on hotspot), raising an issue at https://github.com/eclipse-openj9/openj9/issues is potentially all you need to do.  The openj9 community should be able to recreate and debug the failure themselves - if they need more help they will ask for it.

For hotspot failures and openj9 openjdk failures (or if you just want to do more failure analysis yourself) some more digging will be needed........

### Step 3 - Try to establish the reason for the failure

These are the most likely candidates
1. The test environment (machine setup (or lack of))
2. The test case code
3. The jdk code

These are less likely candidates
1. Corrupt / incomplete jdk build (e.g. executables not executable, excutables not signed, packages 'missing').

#### Is it the test environment?

- Does the test pass on one machine but not another?
- Does the test pass on one operating system but not another?
- Does the test pass on one operating system version but not another?

If it is the test environment, update the issue with what you know - 'Fails on machine x but passes on machine y', 'Fails on AIX only', 'Fails on s390x linux, passes on linux on other architectures'.

Unless you have machine access you will probably need help from the infrastructure team to resolve and fix a machine setup issue.

#### Is it the test case?

If a test passes on hotspot but fails on openj9 (or vice versa), then the failure will be due to some sort of difference in behaviour between the implementations.  It may be a bug, or an assumption by the test case that the hotspot (or openj9) behaviour is the only possible behaviour, or the test uses options which are hotspot (or openj9) specific and not valid for the alternative implementation.

Look for the test output which is the reason for the failure - likely to be a message or thrown exception.  If you're lucky it may be obvious what is wrong - but more likely you'll want to rerun the test and add some debugging to pinpoint the cause of the failure.

There are two ways you can rerun a test - on the AdoptOpenJDK / Adoptium Jenkins farm via the 'Grinder' job, or locally on a machine you own.  If you find you need to add debugging to the test case and / or the jdk then working for instance on VirtualBox linux on your laptop may well be the easiest / fastest way to make progress.

##### Rerunning a test in a grinder (https://ci.adoptopenjdk.net/view/Test_grinder/job/Grinder/build):

The 'unit of execution' for a 'functional' or 'system' test is the test target.  The test target can also be used to run openjdk tests, but the openjdk targets run a lot of subtests and it is likely you will want to rerun just a single failing subtest (the output from the failing test is printed to the Jenkins joblog - the output from passing tests is deleted).  To find an individual failing subtest look for 'TEST: ' (note the trailing space) in the Jenkins job log.

1. To rerun a test target, go to the 'Test Selection Parameters' section in the Grinder 'Build with Parameters' page.
- Set 'BUILD_LIST' to functional, openjdk or system according to the test suite the target is in.
- Set 'TARGET' to the target name (e.g. jdk_math, jdk_math_0, Minimix_5m)
- Click the 'Build' button

2. To rerun an individual openjdk subtest:
- Set 'BUILD_LIST' to openjdk
- Set 'TARGET' to jdk_custom
- Set 'CUSTOM_TARGET' to the path to the test to rerun
  - This is different for jdk8 vs. jdk11+ because jdk directory structure changed after jdk8
  - If the search line for the failing test found in the job log is
`TEST: sun/security/krb5/auto/ReplayCacheTestProc.java`
then for `jdk8` specify the test as `jdk/test/sun/security/krb5/auto/ReplayCacheTestProc.java`
and for `jdk11+` specify the test as `test/jdk/sun/security/krb5/auto/ReplayCacheTestProc.java`
(the top level 'jdk' and 'test' directories are reversed).

3. To make changes to the test case code and then rerun via a grinder:

The test case code can be found on github:
- openjdk tests: https://github.com/adoptium/jdk8u/, https://github.com/adoptium/jdk11u/, etc.)
- functional tests: https://github.com/eclipse-openj9/openj9/
- system tests: https://github.com/adoptium/aqa-systemtest, https://github.com/adoptium/STF

In many cases you will want to add some code to get more information from the failing test (e.g. to print lines to track the test execution path or print the value of certain variables at various points during execution).  The general principle for this is to fork your own copy of the git repository containing the test, create a branch for your changes, make the changes, and run the amended test.

- Fork the repository containing the test
- Clone your fork locally (e.g. to your laptop)
- Create a branch for your changes (e.g. fix_ReplayCacheTestProc)
- Make some changes in your branch
- Commit your changes and push them to your fork
- Rerun the test in a Grinder, this time telling the Grinder job to use your fork / branch
  - In the 'Test Repositories Parameters' section of https://ci.adoptopenjdk.net/view/Test_grinder/job/Grinder/build, change the parameter for the repository you have changed to your fork / branch
    - e.g. If you have changed an openjdk test, you would set JDK_REPO to your github repository fork and JDK_BRANCH to the name of the branch with your changes.
  - When the job runs the tests will be retrieved from your fork / branch rather than the default openjdk source branch
  - If you had added `System.out.println()` statements to the test (and the test fails) your new output will appear in the Grinder job log

#### Is it the jdk?

If you can't find anything wrong with the test case, then maybe the jdk has a bug.

The jdk itself can be forked, modified and built at AdoptOpenJDK by adding build parameters to the jdk build pipeline job - e.g. In https://ci.adoptopenjdk.net/job/build-scripts/job/openjdk8-pipeline/build
- Specify the platform(s) you want to build
- Set 'releaseType' to 'Nightly without Publish' (this means the build will be attached to the job, but not be published to github)
- For the BUILD_ARGS param, specify '-b your_branch -r your_repository' (make sure your_repository is http: protocol, and does not have '.git' on the end)
- Set check whether 'Enable tests' is set correctly.  You probably just want to create a build and use the link to run some Grinders, not to run all the pipeline test jobs after the jdk build job.

### Step 4 - Submit a fix

If after your investigations you have a solution for the failure you can submit your change via a PR or for openjdk changes (need some instructions from openjdk contributors / committers)

Refer to the section below if the 'fix' is to exclude the test from future runs.


### Excluding and reinstating tests

Which tests are executed is controlled via metadata files in the https://github.com/adoptium/aqa-tests repository.
Tests can be excluded in two ways:
1. The makefile targets are defined in playlist.xml files - e.g. https://github.com/adoptium/aqa-tests/blob/master/openjdk/playlist.xml, https://github.com/adoptium/aqa-tests/blob/master/system/mathLoadTest/playlist.xml.  These files basically define the command line to be executed for that target.  An entire target can be excluded by removing it (or commenting it out) from this file.There are also xml tags to specify the target is 'disabled' or should only be run on some platforms / releases.
2. For openjdk tests, where each test target actually runs many tests, there is an additional test exclusion file.  This file is read by the openjdk (jtreg) test harness. Any tests included in it a not executed.  There is a separate file for each jdk release, and a separate file for hotspot vs. openj9 jdks - see https://github.com/adoptium/aqa-tests/tree/master/openjdk/excludes.

Reasons a test may be excluded:
1. A test is no longer valid (e.g. because of a subsequent change to the jdk).
2. The test is not applicable to a platform / implementation.  You will see there are many more test exclusions for openj9 than hotspot because the openjdk tests are sometimes written to test the way openjdk behaves rather than the way openj9 behaves (there is much scope for behaviour differences while still conforming the the Java Specification).
3. The test requires that a fix be made to the jdk, but we do not know when that fix will be done.  Excluding the test means that we don't just keep seeing (and having to triage the same failure) over and over.
The exclusion file requires that the reason for the exclusion (a link to a github issue) is provided for the exclusion.  This means that exclusions can be monitored to see whether they should be reinstated.  In some cases the 'reason' is not really known - the associated issue shows that the root cause of the test failure is not understood.  These exclusions are condidates for more investigation and potentially fixes to test cases or the jdk.

Excluding or reinstating a test is a similar process to amending test cases - the https://github.com/adoptium/aqa-tests repository is forked / branched, changes are made in the branch, and the fork / branch is tested in a Grinder overriding the ADOPTOPENJDK_REPO and ADOPTOPENJDK_BRANCH parameters.  When the changes have been tested they can be merged into the master repository by submitted a Pull request (PR).

### Searching for existing issues

There are a few factors which mean that locating an already existing issue for a test failure is more difficult than one might expect:
1. Because of the many different test suites run at adoptium, the test output format is not consistent across all the tests
2. The different jdk implementations (hotspot, openj9) error messages are different
3. Issues are raised by a variety of community members, one person may focus on one piece of information, another on a different (equally useful) piece.
4. There are a number of different repositories where an issue might have been raised.

If you can't find an issue after trying these options then raise a new one. If it turns out there really was an issue which you missed then eventually one or the other of them will become non-reproducible when a fix is made!

1. Search for the failing test target
- More useful for 'functional' and 'system' test failures than 'openjdk', since for openjdk tests a single test target runs many tests. (This means that if the target is mentioned in an issue it could well apply to a different test. Also raisers often mention only the failing test in the issue, not the test target.)

2. Search for the openjdk failing test
- For 'openjdk' test failure, search for the name of the failing test (its output is printed to the Jenkins job log with a heading "TEST: name_of_test"). This is the best search for these test failures.

3. Search for text output in the failing job
- Sometimes the same failure may affect a number of tests, in which case a specific test may not be mentioned (and if it is, may happen to have been a different test that the failure you're looking into).
- Some examples of when this might be the best serach
- `No space on device`
- `Unable to connect to socket nnn`
- `Expected dump file not found`
- jdk crash report. Particularly for openj9, when the jdk terminates abnormally a crash dump summary is written to the joblog. The same crash might occur when running different tests, so searching for the same (or a very similar) summary might identify the issue.  An example issue for such a crash: https://github.com/eclipse-openj9/openj9/issues/12751

Some search 'techniques':
1. TRSS has a 'Possible issues' action button (available in the 'Test suite analysis' view (click the coloured block for the job in the 'Grid' view)
- It doesn't cover all the search patterns listed above yet (primarily uses `test target`), but it does search multiple repositories at once.
2. Search directly in github
- The issues in individual repositories can be searched using the github web interface. github search is not necessarily intuitive (a search for `replaycachetestproc` will find an issue which mentions test `sun/security/krb5/auto/ReplayCacheTestProc.java` but a search for `replaycache` will not). It is possible to search multiple git repositories at once - see https://docs.github.com/en/github/searching-for-information-on-github/searching-on-github/searching-issues-and-pull-requests
3. Internet search engine
- Also works!
- Good for finding issues in the openjdk bugs database (https://bugs.openjdk.java.net/projects/JDK/issues).