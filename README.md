[Introduction](https://github.com/krebor/DevOpsLab/#introduction) <br />

[Assignment](https://github.com/krebor/DevOpsLab/#assignment) <br />

[Topology](https://github.com/krebor/DevOpsLab/#topology) <br />

[Configuration](https://github.com/krebor/DevOpsLab/#configuration) <br />

  - [TeamCity](https://github.com/krebor/DevOpsLab/#teamcity) <br />
  - [Ansible](https://github.com/krebor/DevOpsLab/#ansible) <br />
  - [Docker](https://github.com/krebor/DevOpsLab/#docker) <br />

[Final Overview](https://github.com/krebor/DevOpsLab/#final-overview) <br />

## Introduction

**DISCLAIMER: This project is Work In Progress and is subject to change.**

This practice lab was designed with the goal of showcasing and teaching basic DevOps principles and technologies. 

This lab involves creating three virtual machines - two Windows machines and one Linux machine. Virtualization software used in this lab is Oracle VirtualBox and the recommended system requirements are at least 16 GB of RAM, quad core processor and 100 GB of hard disk space. Lower specs could be used if using Windows Server images for Windows machines. Better specs are always welcome. Configuration of VM's in VirtualBox won't be covered in this lab, but more detail can be found [here](https://www.virtualbox.org/manual/).

## Assignment

Set up TeamCity on the first Windows VM and Ansible/AWX on your Linux VM. Afterwards, create a configuration which will allow an automatic build of sample Windows service on TeamCity server from GitHub source, then deploy the build over Ansible server to the third (Windows/Client) virtual machine. Two administrator accounts need to be created on the client VM, and the deployed service needs to be executed in isolation under their respective accounts (simulating two destination servers). To recap:

1. Connect TeamCity to GitHub repository and perform build through TeamCity
2. Connect TeamCity to Ansible
3. Deploy build via Ansible to client VM as two isolated Windows services, which are run by separate Administrator accounts

Sources: <br />
https://www.jetbrains.com/teamcity/ <br />
https://github.com/MonoSoftware/sample-windows-service

## Topology

<img src="/assets/devops_topology.png" width="300">

VM01 - Teamcity <br />
VM02 - Ansible <br />
VM03 - Client <br />

## Configuration

### TeamCity

TeamCity is a Continuous Integration (CI) tool which, among other things, enables you to build software from source code, adequately test it and prepare it for deployment with a streamlined set of tools and processes. TeamCity Server is serving as a central control point for your TeamCity environment, while TeamCity Agents are used to perform the actual work of building/testing software. Since this is a simple lab, a single machine will serve as both Server and Agent - it is advised to deploy these roles on different machines in more complex environments.

Start by downloading and installing TeamCity Server on your first Windows VM. For this lab, all installation settings can remain on default selections. Once the installation completes, open a web browser and type `localhost:8111`. If you changed the default port during installation to something other than 8111, please use this port. Accept the license agreement and for database type select **Internal (HSQLDB)**. If you did not change the default selection for authentication during installation, you should be prompted to set up a password for the Super User local system account. After logging in, you will be greeted by the TeamCity WebUI.

We now need to create a new project (named **Sample Windows Service**) and link it to the relevant [GitHub repository](https://github.com/MonoSoftware/sample-windows-service).

After performing these steps, we can now proceed with build configuration. We will name the first build configuration within our project "**Build**" and will proceed to edit this configuration. You should notice that the VCS (Version Control System) details are already populated, as was specified on project creation. 

Next thing to note is the **Build Step** section - here we can see that TeamCity automatically detected a number of possible build steps by looking at the files in the linked GitHub repository. There should be four build steps automatically provided by TeamCity, two of which are **.bat** scripts used for installation/uninstallation of the relevant service. We will use only one provided build step - a .NET build step (using a .NET runner) which will use **msbuild** tools to perform the build. You should disable the other build steps:

<img src="/assets/build_steps.png" width="800">

Also, notice that the **Triggers** configuration is already populated by a VCS trigger, and the following parameter: `+:*`. This means that any addition to the linked GitHub repository will trigger a new build.

The remaining enabled .NET build step should already be correctly configured, but before we actually run the build, we still have a few more configuration items to update. First of these is **Arctifact Paths** within **General Settings** of the build configuration. If you don't specify this, the build will not produce artifacts in an expected way. TeamCity has special syntax for defining such configuration, which you can explore by looking into official TeamCity documentation. To complete this part of configuration, please enter the following:
```
+:**/* => target_directory.tar
-:*.gitattributes => target_directory.tar
-:*.gitignore => target_directory.tar
-:*.sln => target_directory.tar
```
In this case we opted to archive our build artifacts within a **.tar** archive named **target_directory**.

We are now ready to attempt running our build, which we can do by selecting **Run** in the upper right corner. After you select **Run**, an available agent will start working on the build and will most likely fail due to missing dependencies. This should be no cause for concern since these dependencies can be resolved easily by installing required software or modifying system environmental tables - in my case, I had to install the required .NET runtime, MSBuild Tools version and also update the PATH environmental variable to include the relevant **dotnet** executable path. After resolving all dependency issues, we should now be able to perform a clean build:

<img src="/assets/successful_build.png" width="500">

Having completed our build, we now need to push it to the Ansible server for further deployment. To achieve this, we will introduce a second build configuration in our TeamCity project, which we will name **Deploy**. Only one build step will be configured in this part and it will utilize the **SSH Upload** runner. For this to work, we also have to generate an SSH key pair, private and public. [Here](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement) is more info on SSH key pair generation. We will add the generated private key to TeamCity by visiting settings of our project, and adding the relevant private key. The public key will have to be added to Ansible server's **.ssh** directory, within the **authorized_keys** file.

Now we can proceed with configuring the **SSH Upload** build step within our **Deploy** build configuration. Here is a short overview of the configuration:

<img src="/assets/ssh_upload.png" width="500">

**Target** represents the destination server (in this case Ansible server) and the directory where we want the artifacts uploaded. You can leave protocol on **SCP**, then enter select the SSH key you added earlier from the dropdown. Credentials for relevant account on Ansible server should be entered. Please note that in this example the root account was used - this is a bad security practice and you would usually create and use a dedicated account for this type of access. We opted for the root account in this lab environment. Lastly define **Path to sources** which should point to the location of our artifact(s). In this case I entered **target_directory.tar**.

Last but not least, we also have to define a dependency for our **Deploy** build configuration step - after all, we want the SSH Upload to be performed only after a successful build. Go to **Dependencies** and here add a new artifact dependecy. **Depend on** should be populated by the build configuration named "**Build**" and the **Artifact rules** should reflect the source and destination of the relevant artifact - in our case this parameter is populated like this: `target_directory.tar => target_directory.tar`.

Now let's try running our Build and Deploy:

<img src="/assets/successful_push.gif" width="800">

Finally, let's check our destination server (Ansible):

<img src="/assets/ssh_upload_confirm.png" width="500">

Great news, the artifacts archive was successfully transfered to the Ansible server and is ready to be further processed/deployed!

**-- NEXT ENTRIES ARE WORK IN PROGRESS AND DO NOT OFFER A COMPLETE SOLUTION --**

Last piece of configuration which we need to perform on the TeamCity server is to configure an SSH Exec build step in our **Deploy** build configuration, which will execute the required bash script and/or Ansible commands, which will enable further deployment to the Client machine.

### Ansible

Now that we have the build artifacts, we need to create an Ansible Playbook which will perform the transfer of the archive to the Client machine, unarchiving of the file and running relevant Docker commands on the Client machine (`docker build`, `docker run`), which will leverage a pre-configured Dockerfile The required configuration can be created and tested on the Ansible Linux machine, then implemented within TeamCity SSH Exec build step.

We can start our configuration by installing Ansible - this can be done by using the `pip` Python package installer, or your Linux distribution's native package manager. Recommended way is using `pip`, since this method will provide the latest packages.

After installing Ansible, it's time to configure our Inventory file - here we will define all client machines which Ansible will administer. Make sure to create the file within the same directory where your Playbooks and configuration files will be stored. The syntax of the file is simple, which is a list of IP addresses and/or hostnames of the machines you wish to be managed by Ansible.

Next it's time to create an Ansible configuration file. Here we will define global Ansible settings which will enable us to shorten our Ansible commands and make overall quality of life improvements. Name the file **Ansible.cfg** and populate it as needed. Example:

```
[defaults]
inventory = inventory
private_key_file = ~/.ssh/id_rsa
remote_user = root
```
This simple file outlines the structure of the file and some basic settings we can configure. In this case we defined which file will be used as an Inventory file (file we created earlier and named "**Inventory**"),  specified the location of a relevant SSH private key file and configured a default user when connecting to remote hosts (in this case **root**).

Finally, we have to configure a Playbook, which is a YAML file containing the necessary tasks we want to be performed. An example YAML file:

```
---

- hosts: all
  become: true
  tasks:
  
  - name: transfer file
    copy:
      src: /home/krebor/ansible_tutorial/archive.tar
      dest: /root/sample_folder

  - name: unarchive file
    unarchive:
      src: /home/krebor/ansible_tutorial/archive.tar
      dest: /root/sample_folder
      
  - name: build image
    community.docker.docker_image:
      build:
        path: ./WindowsService
      name: windows_service
      source: build
      
  - name: run container
        docker_container:
          name: windows_service_container
          image: windows_service:latest
          state: started
          recreate: yes
          detach: true
          ports:
            - "8888:8080"
```
### Docker

Reference material: https://learn.microsoft.com/en-us/dotnet/core/docker/build-container

Docker Desktop has to be installed and started on the Client Windows machine. The relevant Docker image for Windows has to pulled - proposal in this case is to use **windows/servercore** Docker image. Administrator accounts should be created, Dockerfile configured for Windows Service deployment, and the two containers should be initialized by the Ansible Playbook/ad-hoc command list. Sample Dockerfile:

```
FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

COPY ["WindowsService/bin/Release/", "/Service/"]

WORKDIR "C:/Service/"

RUN New-Service -Name "SampleWindowsService" -BinaryPathName SampleWindowsService.exe; \
    Set-Service -Name "\"SampleWindowsService\"" -StartupType Automatic; \
    Set-ItemProperty "\"Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\SampleWindowsService\"" -Name AllowRemoteConnection -Value 1
    sc config SampleWindowsService obj= DOMAIN\User password= password

ENTRYPOINT ["powershell"]
CMD Start-Service \""MyWindowsServiceName\""
```

## Final Overview

What did we achieve in creating this lab? In short, we enabled an automatic build and deployment of newly commited source code, and even if we did not directly implement any tests, we saw how this could also be implemented as one of the build configurations within TeamCity. We leveraged our knowledge of virtualization and explored new concepts like TeamCity Server configuration and build/deploy process, Ansible configuration automation and Docker containerization. Hopefully this was a valuable experience to spark further study and self-development.
