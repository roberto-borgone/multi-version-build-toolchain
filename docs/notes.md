# Notes
## Goal
The goal is to have a tool able to build many types of artefacts starting from a source folder. This tool should give to the user the possibility to easly (by using a single tool) package its application in a varaiety of different formats from a simple executable file to full fledged VMs.
## How
I was thinking about a sw that works on top of the Gradle building and automation tool (https://docs.gradle.org/current/userguide/userguide.html).

Gradle targets many types of applications (JVM-based, native and many other) and its functionalities can be extended via new plug-ins. Hence, for the “building” step my tool should just configure Gradle in order to properly build the specific type of project (like a sort of setup wizard). Moreover continuous integration could be achived by the interaction with GitHub Actions (this would tie the tool to the use of GitHub as code repository) by letting my tool create all the necessary pipeline files for building the the project. Alternatively it would also be possible to use Jenkins (Gradle already has some interactions with Jenkins). A command like:
```
toolname init
```
should create:
- the Gradle folder structure, for instance for a Java/Kotlin project would be something like this (the important thing is the presence of the <mark>build.gradle</mark> file which should be automatically filled by the tool in order to properly build the particular type of project, or at least it should provide the starting point for it)
```
.
├── build.gradle
└── src
    └── main
        ├── java
        │   └── HelloWorld.java
        └── kotlin
            └── Utils.kt
```
- a <mark>toolname.yaml</mark> file describing the type of project contained in the folder and it should have a section for every type of artefact the user wants to create. This file should be edited by the user. Each section could contain additional informations to customize the final artefact of that type (maybe allowing to create a VM that uses an FPGA or a container with a particular networking setup... ). The format of this document has to be defined and will determine the actual functionalities my tool will be able to deliver.
- a git folder connected to a github remote repo and a <mark>.github</mark> folder containing the required CI pipelines or the <mark>Jenkinsfile</mark> depending on what I will decide to use for implementing CI.

Given the complex nature of a software project and the endless list of purposes it could have, I think that letting the user decide what are the objectives he wants to achive when packaging his application (selecting them among the supported alternatives the tool provides, which can always be extended) would be more effective (and easy to characterize) than trying to guess the possibile artefacts that can be generated from a source folder, however this feature coud be implemented in the future.

The packaging phase should contain the main logic of this tool, I imagine that internally I will keep a catalog of templates (Dockerfiles, Libvirt XMLs, CI/CD pipelines, etc..) divided per type of artefact and per type of project. These templates would be customized by the tool depending on the informations the user has put in the <mark>toolname.yaml</mark> description file. All these artefacts would need to be automatically generated and stored in a remote repository to be used for the deployment (Continuous Delivery), this could be again possible using the CI/CD tools above mentioned.
