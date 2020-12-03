* *Builds* [![Build
Status](https://travis-ci.org/IASI-SAKS/groucho.svg?branch=master)](https://travis-ci.org/IASI-SAKS/groucho)

# GROUCHO

Building
-------
GROUCHO is (mainly) a Maven project. Build and install it with `mvn install`. In order to correctly build GROUCHO, the variable `JAVA_HOME` has to be set, for more details see the section [Java Home](https://github.com/IASI-SAKS/groucho#java-home).

GROUCHO relies on an instrumented JVM (provided by [CROCHET](https://github.com/gmu-swe/crochet)) that will be located in `groucho-crochet/target/jre-inst/`.

Some O.S. libraries may be required in order to build and run GROUCHO. The detailed list can be found in the file `.travis.yaml`.

Java Home
-------
Remember that the variable JAVA_HOME has to be defined and properly set.
Possible hints are:
 * Linux: ```export JAVA_HOME=`dirname $(dirname $(readlink -f $(which javac)))` ```
 * Mc OS: ```export JAVA_HOME=$(/usr/libexec/java_home)```

About the Java Instrumentation
-------
For safety reasons, GROUCHO does not apply its instrumentation on the whole set of Java classes in the class path. More specifically, and in addition to all the classes that CROCHET does not instrument, any agent built from inheritance of the ``it.cnr.iasi.saks.groucho.instrument.AbstractClassTranformer``  does not apply to the instrumentation the classes belonging to the following packages:
 * ``java.*``
 * ``sun.*``
 * ``it.cnr.iasi.saks.groucho.*``
 * ``org.junit.*``
 * ``junit.framework.*``
 * ``org.apache.maven.*``
 
If for some reasons you may need to programmatically exclude some package/class form the instrumentation, please modify the class ``it.cnr.iasi.saks.groucho.instrument.AbstractClassTranformer`` and build the project again.
For example during some local test under Eclipse we had to add the following package to the list of classes to be locally ignored:
 * ``org.eclipse.jdt.internal.junit.* ``
 
An easier alternative is to configure the list of classes subject to exclusion by setting a specific property (e.g. as an entry of a property file). The following example:

`` groucho.transformer.disable.classesList=org/objectweb/asm,com/fasterxml/jackson`` 

shows how to disable the classes in both the packages:
  * ``org.objectweb.asm.*``
  * ``com.fasterxml.jackson.*``
  
 Most of the times when the JVM notifies an error like:
 ``Java.lang.linkage error when instrumenting XXX``
 it is possible a class that have been already loaded (and instrumented) is going to be processed again. If the class ``XXX`` is really not relevant for the Invivo testing campaign, thus it may be useful to exclude it as described above.

How to Enable Invivo Tests
-------

Each method that could be subject to In Vivo testing must be annotated as `TestableInVivo`

```java
	@TestableInVivo(invivoTestClass = "it.cnr.iasi.saks.groucho.invivotests.DummyInvivoTest",
			invivoTest = "invivoTestMethodName")
	public void thisIsFoo() {
		/*
		 * To Something here
		 */
	}
```

In case the source code of a class is not available for modification, the injection can be specified by means of a JSON record. An example is reported in:
 * [modelResource.json](https://github.com/IASI-SAKS/groucho/blob/master/groucho-lab/src/test/resources/modelResource.json)
 * [testingConf.properties](https://github.com/IASI-SAKS/groucho/blob/master/groucho-lab/src/test/resources/testingConf.properties) by setting the property `groucho.lab.intrument.jsonFile` to the path of the file of the JSON record
 
In order to apply the annotations from a given JSON report, please refer the following example shows:
 * the specific GROUCHO agent [`${groucho-lab.build.directory}/${groucho-lab.build.finalName}.jar`](https://github.com/IASI-SAKS/groucho/tree/master/groucho-lab/src/main/java/it/cnr/iasi/saks/groucho/lab/instrument) responsible for the injection
 * how to enact such an agent within a [pom.xml](https://github.com/IASI-SAKS/groucho/blob/master/groucho-lab/pom.xml#L274-L276)

About QA Aspects
-------
Some quality gates are defined and monitored by means of SonarCloud and Jacoco. As GROUCHO is a multi-module maven project, there are few
issues important to remember:
* Currently the token credential for Sonar has been set in the ``SONAR_TOKEN`` environmental variable from the Travis-CI UI 
* The test for [State Carving](groucho-core/src/test/java/it/cnr/iasi/saks/groucho/carvingStateTests/) have been disabled during the QA analysis. This configuration is currently needed even if the tests pass on a "regular" build. Indeed, the injection by Jacoco will cause these tests fail because it modifies the result of the tests so that to mismatch their respective expected outcome.
* While configuring MVN it may be the case to modify the arguments passed to the JVM, for example this may happen frequently when using plugins such as ``surefire`` or ``failsafe``. In those cases, Jacoco may wrongly count and report data on the coverage. The solution is to properly configure ``surefire``/``failsafe`` in order to interact with the ``jacoco:prepare-agent`` plugin. For example do not forget to append new arguments my referring the MVN variable ``@{argLine}``. Further details from the [official documentation](https://www.eclemma.org/jacoco/trunk/doc/prepare-agent-mojo.html). Please note that it is strongly recommended to 
always include in the ``POM`` an empty definition of the property argLine. Indeed if ``@{argLine}`` is used but never initialised thus surfire/failsafe causes the build to fail.
* Jacoco does not really support the analysis of multi-module projects. The work-around is to:
   1. create an artificial module depending from all the others subject to analysis that will actually host the reports
   1. to properly configure all the modules so that to redirect the analysis in such an artificial module.

   Within GROUCHO the module [groucho-sonar](groucho-sonar) has such intent. The followed documentation is:
    * [Maven Multi-Module Builds](https://github.com/jacoco/jacoco/wiki/MavenMultiModule#maven-multi-module-builds)
    * [Multi-module Apache Maven example](https://github.com/SonarSource/sonar-scanning-examples/tree/master/sonarqube-scanner-maven/maven-multimodule)

How to Launch some Experiments
-------
 * Apache JCS: Launch the experiment with the original test cases from Apache JCS
 ```bash
 cd groucho-lab
 mvn -PjcsExperimentsProfile clean verify
 ```
 * OpenSymphony OSCache: Launch the experiment with the test cases independently developed at USI 
 ```bash
 cd groucho-lab
 mvn -PosCacheExperimentsProfile clean verify
 ```
 * OpenSymphony OSCache: Generate new tests by means of Evosuite
 Generate the new tests only if needed, and be carefull to do not overwrite pre-exiting test turned to be launched invivo.
 As first, copy the source code of the CUTs inside the folder `groucho-lab/src/extra/java`. Then customize the properties of the Maven profile [`generateTestStubsProfile`](https://github.com/IASI-SAKS/groucho/blob/master/groucho-lab/pom.xml#L458). Finally, execute the following commands:
 ```bash
 cd groucho-lab
 mvn -PgenerateTestStubsProfile evosuite:generate
 mvn -PgenerateTestStubsProfile evosuite:export
 ```
  The new test will be exported into `groucho-lab/src/extra/test`. Avoid to add this folder in the repository. Turn these test generated for offline session into test cases that could launched online (e.g. avoiding new instantiation of the SUT/CUT). Once the conversion is over, copy the test cases into `groucho-lab/src/test/java`.
 
 * OpenSymphony OSCache: Launch the experiment with the test generated by Evosuite 
 ```bash
 cd groucho-lab
TBD
 ```
    
