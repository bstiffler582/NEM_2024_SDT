## Automated Build Pipeline Lab

### Contents
1. [Introduction](#introduction)
2. [Jenkins Installation](#jenkins_install)
3. [Jenkins Initialization](#jenkins_init)
4. [Repository Setup](#repos)
5. [Jenkins Jobs](#jenkins_jobs)
6. [Automation Interface](#automation_interface)
7. [Testing](#testing)
8. [Outro](#outro)

<a id="introduction"></a>

### 1. Introduction

In the industrial automation industry, TwinCAT is uniquely suited for adopting modern software tooling and practices. Requests from customers looking to implement *continuous integration*, *continuous deployment* and *automated testing* with their PLC programs are becoming more and more frequent. The intent of this lab is to illustrate the overall landscape of CI/CD with TwinCAT, as well as to provide a hands-on demonstration of at least one path to realize these workflows with TwinCAT.

<a id="jenkins_install"></a>

### 2. Jenkins Installation

1. [Download and Install JDK 21](https://download.oracle.com/java/21/latest/jdk-21_windows-x64_bin.exe_)
2. [Download and Install Jenkins LTS](https://www.jenkins.io/download/thank-you-downloading-windows-installer-stable/)
3. During install:
    - Point to JDK path @ `C:\Program Files\Java\jdk-21`
    - Select the "Run service as LocalSystem (not recommended)" option

<a id="jenkins_init"></a>

### 3. Jenkins Initialization

Jenkins typically runs as a remote server to act as the delegating service for a development team's build, test and deployment tasks. For this lab, we will keep things as simple as possible and run the Jenkins service right on our local machine.

1. Change service to run as user account
    - 🪟 + `r`, `services.msc`
    - Locate the **Jenkins** service, open properties
    - Log On tab > select "This account"
    - This fixes that "not recommended" step during installation ^^
1. Navigate to [localhost:8080](localhost:8080) in your favorite web browser
2. Copy/paste the initialization password at `C:\ProgramData\Jenkins\.jenkins\secrets\initialAdminPassword`
3. Don't create any more users (continue as `admin`)
4. Install suggested plugins
5. From Dashboard, select **Manage Jenkins**
    - Security -> Agents -> TCP Port for inbound... = "Random"
6. Enable local checkout:
    - Open the file `C:\Program Files\Jenkins\jenkins.xml`
    - Modify the following XML:
```xml
<arguments>-Xrs -Xmx256m -Dhudson.lifecycle=hudson.lifecycle.WindowsServiceLifecycle -jar "C:\Program Files\Jenkins\jenkins.war" --httpPort=8080 --webroot="%ProgramData%\Jenkins\war"</arguments>
```
To
```xml
<arguments>-Xrs -Xmx256m -Dhudson.lifecycle=hudson.lifecycle.WindowsServiceLifecycle -Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true -jar "C:\Program Files\Jenkins\jenkins.war" --httpPort=8080 --webroot="%ProgramData%\Jenkins\war"</arguments>
```
Restart the Jenkins service.

#### Agent

The agent will be responsible for executing our build and test processes. With common software stacks, build agents are often small, transient containers or virtual machines that are quickly deployed as needed and then cleaned up. Since we are building for TwinCAT, we need an agent that has both the TwinCAT realtime and XAE (or Visual Studio).

> There is an implicit agent, "Built-In Node" that runs on the same machine as the Jenkins server. Since we are running everything locally, this is perfectly sufficient to use. We will go through the process of creating and running an agent manually just to demonstrate.

1. "Set up an agent"
2. Give the new node a name, e.g. `TcAgent`
    - Select "Permanent Agent"
    - Remote root directory = `C:\jenkinsAgent`
    - All other node settings default
3. Open agent "Status" page and copy "Run agent from command line (Windows)" command. Something like:
```ps
curl.exe -sO http://localhost:8080/jnlpJars/agent.jar
java -jar agent.jar -url http://localhost:8080/ -secret a94f43e1e787477c9d84... -name TcAgent -workDir "C:\jenkinsWork"
```
4. Open PowerShell as administrator and run the command

We've now manually configured a custom build agent which is running locally and listening for jobs from the Jenkins server. Again, for this lab we can just use the built-in agent. No need to keep this one running - it was just an exercise.

> Imagine a large development team that pumps out several builds of different projects or micro-services daily. They likely need to be able to configure multiple *remote* agents, all with varying environments; different tech stacks, dependencies, build configurations, etc. Think about what the pipelines for the Verl development team might look like...

<a id="repos"></a>

### 4. Repository Setup

Before we create our Jenkins job, we need a way to trigger its execution. Typically, automated builds are triggered by a **push** or **merge** to a remote repository. We are going to create a couple repositories on our local machine. One will act as our 'local', and one as our 'remote'.

> Remote repositories are usually just that: remote. They are hosted online somewhere like Github or Azure DevOps. There is no reason a remote repository can't be hosted right on our local filesystem, though.

Open a PowerShell window and make sure you can run the `git` command. If you get a "git is not recognized..." response, you likely just need to add a `PATH` environment variable. If you have not manually installed Git for Windows, it will still already be installed with Visual Studio in the following location:

```
C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\TeamFoundation\Team Explorer\Git\cmd
```
Now we will create two directories for our local and remote repositories. I will stick mine in a `repos` folder in the root for easy access:
```ps
cd C:\
mkdir repos
cd repos
```
Now, we will create an empty `remote` repository, and clone it to our `local` directory:
```ps
git init --bare remote
```
```ps
git clone remote local
```

Now we have an empty remote master branch (just like e.g. creating a new repository in Github), and we have a local clone as a working directory. Let's create a new TwinCAT project in our local folder, and add a PLC project. There is a `TwinCAT3.gitignore` file included in this repository for you to use with your new project. Via the Git Changes window, we should see Visual Studio tracking our differences between local and remote. Go ahead and commit/push the addition of the TwinCAT project.

> Note: If you observe the contents of the remote folder, you will not see any of your project files, but I promise, they are there. This is how data is stored in remote Git repositories. Git is optimized for textual compression and keeps its own index of what data goes where. These files are sometimes called *blobs*. If you want to test it, feel free to clear the contents of the `local` folder and then re-clone `remote`.

Now that we have a valid remote repository with a project in it, let's create a **Job** in Jenkins.

<a id="jenkins_jobs"></a>

### 5. Jenkins Job

Create a new Job. Call it something like *TcBuild*, and select a project type of "Freestyle Project". You can add a description if you want.

Settings:
1. Source Code Management 
    - Select **Git**
    - Point to the file path of our remote repository `C:\repos\remote`
2. Build Triggers
    - Select **Poll SCM**
    - Schedule value of `H/5 * * * *` (once every 5 minutes)
3. Build Environment
    - Check "Delete workspace before build starts"
4. Build Steps
    - Select **Execute Windows batch command**
    - Enter command `echo job has run!`

Save / Apply the job.

With this configuration, the job will poll our `remote` repository for new commits every 5 minutes. If a change is detected, the latest changes are fetched to the workspace directory (`C:\ProgramData\Jenkins\.jenkins\workspace\[JobName]`) and the script is executed.

If we keep an eye on the Dashboard, sometime in the next five minutes (or so), we will see the server kick off the job. Click the job run and check out the results. Some useful pieces of information:
- The commit message, triggering event and diagnostics info (run time, etc.)
- The Console Output page with a full log of the job
    - (Hopefully) including our echo script
- The project files in our workspace directory

<a id="automation_interface"></a>

### 6. Automation Interface

All the standard pieces for an automated build pipeline are in place, so let's move on to the TwinCAT specific stuff. The most accessible means of automating a TwinCAT project build would be via the Automation Interface PowerShell API. The following script will open the solution (in the background), select the project and perform a build.

 ```ps
$dte = new-object -com "TcXaeShell.DTE.17.0" # XaeShell64 COM ProgId
$dte.SuppressUI = $true # suppress VS interface

# open solution file
echo "Opening solution"
$slnPath = "$pwd\TwinCAT Project\TwinCAT Project.sln"
$sln = $dte.Solution
$sln.Open($slnPath)

echo "Building TwinCAT project"
$sln.SolutionBuild.Build($true)

echo "Exiting..."
$dte.Quit()
 ```

 Copy the script into a file in the root of the repository with the name `tc_build.ps1`. Before committing the changes, modify the job in Jenkins to call the new build script. 
 
 Change:
 ```cmd
 echo job has run!
 ```
 to
 ```cmd
 powershell -ExecutionPolicy Bypass -File "tc_build.ps1"
 ```

 Commit, push, and wait patiently. If everything is sorted, we should see our job kick off and run the Automation Interface script to open and build the project. Since this was done silently (`SuppressUI = $true`), we can verify the success of the job by checking the Console Output page, and the workspace directory for a build output (`_Boot` folder).

 The final piece of the continuous integration pipeline would be to package up our build output for distribution. Navigate back to the job configuration page and scroll all the way down to "Post-build Actions". Add a new action of type **Archive the artifacts**, and enter the following in *Files to archive*:
 ```
 [ProjectName]\_Boot\**\*
 ```
 This is just telling the agent to grab everything from the `_Boot` folder and set it aside from the workspace directory. If we run our job again, we will now see build artifacts as part of the last successful job status. We can navigate these files and download archives for distribution.
 
 The next natural step would be *deployment* of these artifacts to some remote target. Jenkins is capable of this functionality with additional plugins, but that princess is in another castle (for today). It is not a leap to imagine using file transfer utilities and scripts to update remote target boot folders with the generated artifacts.

 > Fun fact: the TwinCAT 4026 Boot folder has been moved to `C:\ProgramData\Beckhoff\TwinCAT\3.1`

<a id="testing"></a>

### 7. Testing

Automated testing is another topic of the software development conversation that has gained a lot of attention over the past several years from TwinCAT users. Customers with exposure to testing libraries and frameworks for common software languages (C#, JavaScript, python) are hoping to be able to implement similar workflows with their PLC development stacks.

>None of us are strangers to comprehensive testing of our PLC programs. However, the mindset of developing software modules *around* their ability to be tested (often called TDD - Test Driven Development) is a relatively novel strategy - especially for "device-level" programmers like us.

Let's add some automated tests to our TwinCAT project. 

>**Unit Tests** are the easiest to describe and implement, because they test an individual *unit* of code and nothing more. Once multiple modules, components or execution paths become involved, it is no longer a *unit* test, and instead something else.

There are a handful of community-developed unit testing frameworks available for TwinCAT that provide a wide range of features. For ease of demonstration, we will use the bare-bones "test framework" library included in this repository.

Clone or download the library project and add it to your solution.

First, bring in the `NEM_TestFramework` project via **PLC -> Add existing item**. We will save this project as a library, install it, and set it up as a *Reference project* in TwinCAT. This is a 4026 feature which allows us to use the project as a library, but also directly debug it from the same solution if necessary. 

The TestFramework project contains Function Blocks to make specifying and running tests easier, but most importantly, it will format the ensuing results in a way that Jenkins can digest:

```xml
<testsuite tests="3">
    <testcase classname="foo1" name="ASuccessfulTest"/>
    <testcase classname="foo2" name="AnotherSuccessfulTest"/>
    <testcase classname="foo3" name="AFailingTest">
        <failure type="NotEnoughFoo">Details about failure</failure>
    </testcase>
</testsuite>
```

Now, we just need an application to test. Let's write some simple, testable PLC code. Create a structure that will act as our "Recipe":
```js
TYPE RecipeData :
STRUCT
	fTemperature        : REAL;
	nAgitatorSpeed      : DINT;
	nMixTimeSecs        : DINT;
END_STRUCT
END_TYPE
```
And then a function which validates the bounds of the recipe values:
```js
FUNCTION ValidateRecipe : BOOL
VAR_INPUT
	Recipe          : REFERENCE TO RecipeData;
END_VAR
VAR
END_VAR
```
```js
ValidateRecipe :=
    (Recipe.fTemperature >= 140.0 AND Recipe.fTemperature <= 185.0) AND
    (Recipe.nAgitatorSpeed >= 10 AND Recipe.nAgitatorSpeed <= 100) AND
    (Recipe.nMixTimeSecs >= 300 AND Recipe.nMixTimeSecs <= 500); 
```
Now we can create some tests to ensure the recipe validation function works as expected. Create a new FB which inherits from the `FB_SimpleTest` block (defined in the library):
```js
FUNCTION_BLOCK FB_RecipeValidTest EXTENDS FB_SimpleTest
VAR_INPUT
	bExpected       : BOOL;
	MockRecipe      : RecipeData := (
                        fTemperature:=140.0, 
                        nAgitatorSpeed:=10, 
                        nMixTimeSecs:=300
                    );
END_VAR
VAR_OUTPUT
END_VAR
VAR
END_VAR
```
We include some initialized mock data to run the test with.
The `Execute` method will actually call our validation function and test the results with an expected value:
```js
METHOD Execute : BOOL
VAR_INPUT
END_VAR
VAR_INST
	bResult		: BOOL;
END_VAR
```
```js
bResult := ValidateRecipe(MockRecipe);
AssertEquals(bExpected, bResult, 'Recipe validation failed');
Execute := TRUE;
```
We will not override any other properties or methods from the base type.

We can now use this block to test multiple recipe validation scenarios. Set this up in `MAIN` to account for just a few different cases:
```js
fbRecipeTest_Valid              : FB_RecipeValidTest(sTestName:='Recipe Valid') := 
                                    (bExpected:= TRUE);
fbRecipeTest_InvalidLoTemp      : FB_RecipeValidTest(sTestName:='Recipe Invalid Lo Temp') := 
                                    (bExpected:= FALSE, 
                                    MockRecipe:=(fTemperature:=139.9));
fbRecipeTest_InvalidHiTemp      : FB_RecipeValidTest(sTestName:='Recipe Invalid Hi Temp') := 
                                    (bExpected:= FALSE,
                                    MockRecipe:=(fTemperature:=185.1));
```
All the test setup is taken care of right in the declaration. We just need to instatiate the test runner block and pass our test cases in:
```js
fbTestRunner                    : FB_TestRunner;
bInit                           : BOOL;
```
```js
// run once
IF NOT bInit THEN
    fbTestRunner.AddTest(fbRecipeTest_Valid);
    fbTestRunner.AddTest(fbRecipeTest_InvalidLoTemp);
    fbTestRunner.AddTest(fbRecipeTest_InvalidHiTemp);
    bInit := TRUE;
END_IF
fbTestRunner.Execute();
```
The test runner will handle calling the `Execute` method of our test blocks, and saving off all the results to an XML file on the root of our local machine as `\testresults.xml`. Feel free to modify the mock data or expected values to force a test failure.

> Be sure to read in the runtime configuration and set appropriately for your local target.

Finally to tie everything together:
- Modify the build script
    - Activate configuration and run
    - Copy the test results output to Jenkins workspace
```ps
$dte = new-object -com "TcXaeShell.DTE.17.0" # XaeShell64 COM ProgId
$dte.SuppressUI = $true # suppress VS interface

# open solution file
echo "Opening solution"
$slnPath = "$pwd\TwinCAT Project\TwinCAT Project.sln"
$sln = $dte.Solution
$sln.Open($slnPath)

echo "Building TwinCAT project"
$sln.SolutionBuild.Build($true)

echo "Activating Configuration"
$systemProject = $sln.Projects.Item(1)
$systemManager = $systemProject.Object
$systemManager.ActivateConfiguration()

echo "Restarting TwinCAT runtime"
$systemManager.StartRestartTwinCAT()

echo "Copying test results"
Start-Sleep 5 # give the test process some time...
Move-Item -Path C:\testresults.xml -Destination $pwd\testresults.xml

echo "Exiting..."
$dte.Quit()
```
- Add one more **Post-build Action** to the Jenkins job:
    - Type *Publish JUnit test result report*
    - *Test report XMLs* value: `testresults.xml`

Commit and push your changes. Either wait for the SCM Poll to trigger the job, or manually initiate it. Watch the output and ensure the test results are moved to the workspace directory. You should now be able to view the results within Jenkins.

<a id="outro"></a>

### 8. Outro

We have demonstrated automating the build and test process of a TwinCAT project using Jenkins, PowerShell and the Automation Interface API. With this exposure, hopefully you have gained familiarity with DevOps tooling and terminologies, as well as the continuous integration workflow. From the TwinCAT perspective, realizing this workflow in other comparable tools (e.g. Azure DevOps) would look very similar.
