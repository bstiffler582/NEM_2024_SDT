## Automated Build Pipeline Lab

### Contents
1. [Jenkins Installation](#jenkins_install)
2. [Jenkins Initialization](#jenkins_init)
3. [Create an Agent](#create_agent)

<a id="jenkins_install"></a>

### 1. Jenkins Installation

1. [Download and Install JDK 21](https://download.oracle.com/java/21/latest/jdk-21_windows-x64_bin.exe_)
2. [Download and Install Jenkins LTS](https://www.jenkins.io/download/thank-you-downloading-windows-installer-stable/)
3. During install:
    - Point to JDK path @ `C:\Program Files\Java\jdk-21`
    - Select the "Run service as LocalSystem (not recommended)" option

<a id="jenkins_init"></a>

### 2. Jenkins Initialization

Jenkins typically runs as a remote server to act as the delegating service for a development team's build, test and deployment tasks. For this lab, we will keep things as simple as possible and run the Jenkins service right on our local machine.

1. Navigate to [localhost:8080](localhost:8080) in your favorite web browser
2. Copy/paste the initialization password at `C:\ProgramData\Jenkins\.jenkins\secrets\initialAdminPassword`
3. Don't create any more users (continue as `admin`)
4. Install suggested plugins
5. From Dashboard, select **Manage Jenkins**
    - Security -> Agents -> TCP Port for inbound... = "Random"

<a id="create_agent"></a>

### 3. Create an Agent

The agent will be responsible for executing our build and test processes. With common software stacks, build agents are often small, transient containers or virtual machines that are quickly deployed as needed and then cleaned up. Since we are building for TwinCAT, we need an agent that has both the TwinCAT realtime and XAE (or Visual Studio).

1. "Set up an agent"
2. Give the new node a name, e.g. `TcAgent`
    - Select "Permanent Agent"
    - All other node settings default
3. Open agent "Status" page and copy "Run agent from command line (Windows)" command. Something like:
```
curl.exe -sO http://localhost:8080/jnlpJars/agent.jar
java -jar agent.jar -url http://localhost:8080/ -secret a94f43e1e787477c9d848c464dad5d6db7ff53c37e14be02b42401e264db7d17 -name TcAgent -workDir "C:\jenkinsWork"
```
4. Open PowerShell as administrator and run the command

Our local machine is now configured as an agent, and is listening for jobs from Jenkins.

---

<a id="introduction"></a>

### 1. Introduction

MQTT tooling in TwinCAT is particularly simple and comprehensive. In this lab, we will demonstrate basic usage of the MQTT client libraries to publish and subscribe to a web-hosted broker. You will also gain exposure to JSON handling and the use of the `ANY` type for writing generic, reusable functions.