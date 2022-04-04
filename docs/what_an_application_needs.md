# APPS

## Executable level

Apps have dependencies. Modules/libraries must be taken into account at build time, at test and run time (run time dependencies are taken into account at artefact level).

Gradle is able to provide dependencies management, but would it be possible to automatically configure build.gradle / settings.gradle files without user intervention? (parsing the source code maybe?)

## Artefact level

### Self contained artefacts
Once the app is built, when it has to be packaged in self contained artefacts such as containers and VMs, it could be necessary for the user to also include other apps in the package. Each of these applications must be in some way included in the artefact at its creation by either:
- compiling the source code
- copying an executable
- executing an apt get command
- cloning a repository and following its installation guide
- ...?

The final artefacts could need to be accessible from the outside or to store data, therefore they need to export:
- network ports
- mount points

### Simple executables

For artefacts like simple executables, their installation might require to act directly on the target machine, this could be done by using Ansible, therefore producing the right playbook for the job. More or less it should be similar to how to install the same application in a container, even the configuration (like which APIs the app should contact, which user/psw/key it should use, etc...) can be parameterized using Ansible.

## WHAT IS POSSIBLE AND WHAT IS NOT

I don't think that I would be able to guess all the application needs, maybe the dependencies at executable level (see automake) can be deducted from the source code but there are other dependencies external to the application and its code that cannot be guessed (you can't guess if the user want to put a proxy in the same container of the application by reading the application source code). I must ask to the user to explicitely tell the tool which are the dependencies and also how to install them (some simple cases could be automatically handled without having to write the installation commands). So my tool would neeed a descriptor file of the artefact, although this would just move the problem from "write a Dockerfile" (for instance) to "write a descriptor file for the tool". On the other hand it could be convenient for the user to have a single tool that lets you produce all sorts of different artefacts by writing its description only once, with a single syntax to learn. 