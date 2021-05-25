## Triaging

### Step 1. Find failures

- Find a build in TRSS (https://trss.adoptopenjdk.net/)
- Use the various TRSS views and / or Jenkins job logs to locate failing tests
- Every test belongs to a test target (which is the level at which TRSS analysis is done)
- For openjdk tests, there are many tests within a test target. If the test target has failed you need to look for "TEST: " in the job log to find the failing test(s).
- Do a quick review of all the tests for a build - your looking for an overall appreciation of what is failing
-- Are the same tests failing everywhere (may indicate an infrastructure issue, may indicate a cross platform regression, may indicate a new / changed test which doesn't work, or doesn't work in our environment) or are they unique to a platform / release (indicates machine / platform specific setup issues, platform / release specific test case issues or actual jdk issues)
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
-- Not all tests run on both hotspot and openj9. In particular, many more 'functional' tests are run on openj9 (thety are the functional tests written by openj9 to test openj9), and many more 'openjdk' tests are run on hotspot (they are the tests written by openjdk to test openjdk and may rely on openjdk / hotspot behaviour).  So if a test fails on openj9 or hotspot but does not fail on the other implementation it might be because it is not run on the other implementation.

Is the test new?
- Useful to know - it never have been run in the failing environment - it might just not work.

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
- If it's obviously and eclipse-openj9 issue (doesn't fail on hotspot) - raise in https://github.com/eclipse-openj9/openj9/issues
- If it's a systemtest issue, raise in https://github.com/adoptium/aqa-systemtest

If it is known:
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
1. Corrupt / incomplete jdk build (e.g. executables not executable, excutables not signed, packages 'missing).

#### Is it the test environment?

- Does the test pass on one machine but not another?
- Does the test pass on one operating system but not another?
- Does the test pass on one operating system version but not another?

If it is the test environment, update the issue with what you know - 'Fails on machine x but passes on machine y', 'Fails on AIX only', 'Fails on s390x linux, passes on linux on other architectures'.

Unless you have machine access you will probably need help from the infrastructure team to resolve and fix a machine setup issue.

#### Is it the test case?

If a test passes on hotspot but fails on openj9, then the failure will be due to some sort of difference in behaviour between the implementations.  It may be a bug, or an assumption by the test case that the hotspot behaviour is the only possible behaviour, or the test uses options which are hotspot specific and not valid for openj9.

Look for the test output which is the reason for the failure - likely to be a message or thrown exception.  If you're lucky it may be obvious what is wrong - but more likely you'll want to rerun the test and add some debugging to pinpoint the cause of the failure.

There are two ways you can rerun a test - on the AdoptOpenJDK / Adoptium Jenkins farm via the 'Grinder' job, or locally on a machine you own.  If you find you need to add debugging to the test case and / or the jdk then for working for instance on VirtualBox linux on your laptop may well be the easiest / fastest way to make progress.

To rerun a test in a grinder (https://ci.adoptopenjdk.net/view/Test_grinder/job/Grinder/build):

To rerun a test target, go to the 'Test Selection Parameters' section in the Grinder 'Build with Parameters' page.
Set 'BUILD_LIST' to functional, openjdk or system according to the test suite the target is in.
Set 'TARGET' to the target name (e.g. jdk_math, jdk_math_0, Minimix_5m)
Click the 'Build' button

The openjdk targets run a lot of subtests and it is likely you will want to rerun just a single failing subtest (the output from the failing test is printed to the Jenkins joblog - the output from passing tests is deleted).  To find an individual failing subtest look for 'TEST: ' (note the trailing space) in the Jenkins job log.

To rerun an individual openjdk subtest:
Set 'BUILD_LIST' to openjdk
Set 'TARGET' to jdk_custom
Set 'CUSTOM_TARGET' to the path to the test to rerun
- This is different for jdk8 vs. jdk11+ because jdk directory structure changed after jdk8
- If the search line for the failing test found in the job log is
TEST: sun/security/krb5/auto/ReplayCacheTestProc.java
then for jdk8 specify the test as jdk/test/sun/security/krb5/auto/ReplayCacheTestProc.java
and for jdk11+ specify the test as test/jdk/sun/security/krb5/auto/ReplayCacheTestProc.java
(the top level 'jdk' and 'test' directories are reversed.

The test case code can be found on github:
openjdk tests: https://github.com/adoptium/jdk8u/, https://github.com/adoptium/jdk11u/, etc.)
functional tests: https://github.com/eclipse-openj9/openj9/
system tests: https://github.com/adoptium/aqa-systemtest, https://github.com/adoptium/STF

It is likely that you will want to add some code to get more information from the failing test (e.g. to print lines to track the test execution path or print the value of certain variables at various points during execution).  The general principle for this is to fork your own copy of the git repository containing the test, create a branch for your changes, make the changes, and run the amended test.

To make changes and rerun via a grinder:
1. Fork the repository containing the test
2. Clone your fork locally (e.g. to your laptop)
3. Create a branch for your changes (e.g. fix_ReplayCacheTestProc)
4. Make some changes in your branch
5. Commit your changes and push them to your fork
6. Rerun the test in a Grinder, this time telling the Grinder job to use your fork / branch
- In the 'Test Repositories Parameters' section of https://ci.adoptopenjdk.net/view/Test_grinder/job/Grinder/build, change the parameter
- So if you had changed an openjdk test, you would set JDK_REPO to you github repository fork and JDK_BRANCH to the name of the branch with your changes.
- When the job runs the tests will be retrieved from your fork / branch rather than the default openjdk source branch
- If you had added Syste,out.println() statements to the test (and the test fails) your new outpt will appear in the Grinder job log

#### Is it the jdk?

If you can't find anything wrong with the test case, then the failure suggests the jdk may have a bug.

The jdk itself can be forked and modified and built at AdoptOpenJDK by adding build parametersd to the jdk build pipeline job - e.g. In https://ci.adoptopenjdk.net/job/build-scripts/job/openjdk8-pipeline/build
1. Specify the platform(s) you want to build
2. Set 'releaseType' to 'Nightly without Publish' (this means the build will be attached to the job, but not be published to github)
3. For the BUILD_ARGS param, specify '-b <yourbranch> -r <yourrepo>' (make sure <yourrepo> is http: protocol, and does not have '.git' on the end)
4. Set check whether 'Enable tests' is set correctly.  You probably just want to create a build and use the link to run some Grinders, not to run all the pipeline tests after the build.

### Step 4 - Submit a fix

If after your investigations you have a solution for the failure you can submit your change via a PR or for openjdk changes (need some instructions from openjdk contributors / committers)

Refer to the section below if the 'fix' is to exclude the test from future runs.


###Excluding and reinstating tests

Which tests are executed is controlled via metadata files in the https://github.com/adoptium/aqa-tests repository.
Tests can be excluded in two ways:
1. The makefile targets are defined in playlist.xml files - e.g. https://github.com/adoptium/aqa-tests/blob/master/openjdk/playlist.xml, https://github.com/adoptium/aqa-tests/blob/master/system/mathLoadTest/playlist.xml.  These files basically define the command line to be executed for that target.  An entire target can be excluded by removing it (or commenting it out) from this file.
2. For openjdk tests, where each test target actually runs many tests, there is an additional test exclusion file.  This file is read by the openjdk (jtreg) test harness. Any tests included in it a not executed.  There is a separate file for each jdk release, and a separate file for hotspot vs. openj9 jdks - see https://github.com/adoptium/aqa-tests/tree/master/openjdk/excludes

Reasons a test may be excluded:
1. A test is no longer valid (e.g. because of a subsequent change to the jdk).
2. The test is not applicable to a platform / implementation.  You will see there are many more test exclusions for openj9 than hotspot because the openjdk tests are sometimes written to test the way openjdk behaves rather than the way openj9 behaves (there is much scope for behaviour differences while still conforming the the Java Specification).
3. The test requires that a fix be made to the jdk, but we do not know when that fix will be done.  Excluding the test means that we don't just keep seeing (and having to triage the same failure) over and over.
The exclusion file requires that the reason for the exclusion (a link to a github issue) is provided for the exclusion.  This means that exclusions can be monitored to see whether they should be reinstated.  In some cases the 'reason' is not really known - the associated issue shows that the root cause of the test failure is not understood.  These exclusions are condidates for more investigation and potentially fixes to test cases or the jdk.

Excluding or reinstating a test is a similar process to amending test cases - the https://github.com/adoptium/aqa-tests repository is forked / branched, changes are made in the branch, and the fork / branch is tested in a Grinder overriding the ADOPTOPENJDK_REPO and ADOPTOPENJDK_BRANCH parameters.  When the changes have been tested they can be merged into the master repository by submitted a Pull request (PR).