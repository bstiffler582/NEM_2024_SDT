## Automated Build Pipeline Lab

### Contents
1. [Introduction](#introduction)
2. [Jenkins Installation](#jenkins_install)
3. [Jenkins Initialization](#jenkins_init)
4. [Repository Setup](#repos)
5. [Jenkins Jobs](#jenkins_jobs)
6. [Automation Interface](#automation_interface)
7. [Testing](#testing)

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

Our local machine is now configured as an agent, and is listening for jobs from Jenkins. Keep this shell window open for the duration of the lab. Don't worry if you accidentally close it - the Jenkins server will detect that the Agent is no longer running, and give you the script again to re-run it.

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
$slnPath = "$pwd\TwinCAT Project\TwinCAT Project.sln" # <- your sln path
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



> Be sure to read in the runtime configuration and set appropriately for your local target.