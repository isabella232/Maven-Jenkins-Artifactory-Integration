#Maven + Jenkins + Artifactory Integration

Working at Sinch has its advantages, but I was recently set up with the task of seperating our application code from our sister comopany and the test framework we use so they can run seperatly. This is great, but for a tech company that has been running for 8 years, this was going to be a big task!

In this quick tutorial, I wanted to show how to get started using Artifactory and #####

Before this, we had the luxury of needing only one dependency, which was not a big overhead to keep and exported JAR right in the project folder. This new seperation  made it obvious that we need any remote server acting as an artifacts storage to get better transparency in code changes and version control. ##Setting up Artifactory
![](images/artifactory.png)

First thing to start with is downloading and installing Artifactory from its [official site](http://www.jfrog.com/open-source/).
We are using a Windows server virtual machine as a base for the Artifactory service to run. To install Artifactory, all you need to do is just open the downloaded archive, go to  `%ARTIFACTORY_HOME%\bin` and execute the file `artifactory.bat`. 

This script searches for the Java executable and runs Artifactory's main class. After installing Artifactory you need to start it’s service either via Windows Services menu or using Command Prompt `(sc start|stop Artifactory)`.

![service settings](images/services.png)

After that Artifactory should be available at `http://SERVER_DOMAIN:8081/artifactory`. The dashboard looks something like this:

![dashboard](images/dashboard.png)

First step is done! 

Next we need to get it integrated into the CI server, which in our case is Jenkins. There is a pretty simple and user-friendly plugin for this purpose called [Artifactory plugin](https://wiki.jenkins-ci.org/display/JENKINS/Artifactory+Plugin). After installing it, you should find a new ‘Artifactory’ section in your Jenkins settings where you need to specify a Artifactory URL and user credentials.

![developer rights](images/developer-rights.png)

Now we have a pipeline which can publish artifacts to the remote Artifactory server. 

The next step is configuring a job which will build our target project and push the compiled JAR to artifactory. 

Create a new job, and in the settings page find the Build section and a specify path to the POM file. Also you will need to add Maven goals to get your project files built. In our case we also would like to synchronize the Jenkins build version with the deployed artifact’s version. This is made via passing Jenkins `BUILD_NUMBER variable to Maven. 

![build](images/build.png)

In the post-build actions section, add deploy artifacts to the Artifactory step and specify that you would like to add your artifacts and check the ‘Deploy Maven Artifacts’ checkbox.

![deploy](images/deploy.png)

And, of course, you need to specify project name and path to your repository.

![source settings](images/source-settings.png)

As a final step, you will need to modify the target project’s POM file. Make sure you have `Maven-compiler-plugin` and `Maven-jar-plugin` in the plugins section.

````
<plugins>
     <plugin>
        <artifactId>Maven-compiler-plugin</artifactId>
        <version>3.1</version>
        <configuration>
          <source>1.6</source>
          <target>1.6</target>
        </configuration>
     </plugin>
     <plugin>
        <groupId>org.apache.Maven.plugins</groupId>
        <artifactId>Maven-jar-plugin</artifactId>
        <version>2.5</version>
        <configuration>
          <archive>
             <index>true</index>
             <manifest>
                <addClasspath>true</addClasspath>
             </manifest>
             <manifestEntries>
                <Implementation-Version>${build.number}</Implementation-Version>
             </manifestEntries>
          </archive>
        </configuration>
     </plugin>
</plugins>
````

it is also important to add groupId, artifactId, verison and packaging to the top-level element of the POM. Please note that we are using `${build.number}` property which we are getting from Jenkins as a parameter when the project is built.

```
<groupId>CommonKeywords</groupId>
<artifactId>CommonKeywords</artifactId>
<version>0.0.1-SNAPSHOT-${build.number}</version>
<packaging>jar</packaging>
```

Bingo! Everything is set up to publish a compiled JAR file to a remote server. All you need to do is save all your changes, push it to your remote repository and press the ‘Start building’ button in your Jenkins job which is connected to Artifactory. 

After the job has finalized all of its steps in the ‘Build history’ section, you should see a link to Artifactory where all the produced artifacts are stored.

![build history](images/build-history.png)

From now on it is possible to use published artifacts in other projects as a Maven dependency. 

All you need to set this up is to add a dependency with the groupId and artifactId you specified in the previous steps. The version of the added dependency you can specify based on the version number which is deployed to Artifactory, e.g.:

```
<dependency>
        <groupId>CommonKeywords</groupId>
        <artifactId>CommonKeywords</artifactId>
        <version>0.0.1-SNAPSHOT-54</version>
</dependency>
```

Basically, Maven searches all its dependencies in the central Maven repository. But we have not pushed our project there and that’s why we have to point Maven to our private Artifactory server. You can go to Artifactory and under ‘Artifacts’ in ‘Tree browser’ it’s easy to find releases or snapshots repository (we specified it in one of previous steps in job settings). 

On this page there is a ‘Distribution management’ section which is exactly where you need to add your POM to get a proper conneciton to Artifactory. Also, if you want to enable snapshots in addition to release versions, you will need to add the appropriate property to your ‘repository’ section in the POM. The other option is to separately define a snapsot and release repository.

```
<repositories>
  <repository>
     <id>CLAESTOR</id>
     <name>CLAESTOR-snapshots</name>
     <snapshots>
                <enabled>true</enabled>
     </snapshots>
     <url>http://your_ip_address:8081/artifactory/snapshots</url>
  </repository>
</repositories>
```

And that’s it!

The following scheme shows how the combination of Maven, Jenkins and Artifactory works in a split team environment to maintain a shared test automation framework. 

![process](images/process1.png) 

![process 2](images/process2.png)


A common workflow working with Artifactory is as follows:

1.  Changes in target project are pushed to Git repository.
2. Jenkins gets latest code from Jit and triggers a Job.
3. Jenkins job uses Maven goals to package a Jar (clean install) and to assign a proper version to the Jar `(-Dbuild.number=${BUILD_NUMBER})`.
4. Jar file gets pushed to Artifactory by Jenkins job (as post-build action). Now the artifact is ready to be downloaded and used as a Maven dependency.
5. Maven dependency (Pom file) in projects which are using target project are updated to the latest version available from Artifactory.
6. Maven downloads dependency from Artifactory and stores it on a local machine `(../.m/repositories)`.

If you have any questions, please leave a comment and I will help you out.

Thanks,

Maryia Tuleika
Test Automation Developer at Sinch
