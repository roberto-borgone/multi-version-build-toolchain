# How to write a Dockerfile

first of all the syntax of a Docker file, then we'll se how the tool could produce it automatically.

## Environment replacement

The evironment variables (created with `ENV` instruction) can be referred in the Dockerfile and they will be replaced with their values. They can be referred using `$variable_name` or `${variable_name}`

```
ENV FOO=/bar
WORKDIR ${FOO}    # WORKDIR /bar
ADD . $FOO        # ADD . /bar
```

Is it also possible to do more sophisticated things like:

- `${variable:-word}` which means: if `variable` is set then use that value otherwise use `word`
- `${variable:+word}` which means: if `variable` is set then use `word` as value, otherwise empty string

<div id="dockerignore"></div>

## .dockerignore

Writing the .dockerignore automatically is not trivial hence I would let the user write it. But here's a description of what it is.
Excludes some files from the context of build. Useful to avoid sending to the docker deamon unnecessary files and possibly mentioning them in some `COPY` or `ADD` instruction. Keep in mind that the root of the context of build is considered in this file as both the root directory and the working directory (writing relative of absolute paths is the same thing). Syntax:

- `*`, means 'any' folder or file `*/temp` matches every file called 'temp' in any immediate subdirectory of the root. Can be also used to complete a name, `temp*` matches also with temporaryfile.txt for instance.
- `**`, matches with any number of directories including zero (the root).
- `?` can be used at the end of a word to indicate a single unknown character, for instance `temp?` matches with 'tempa' or 'temps'
- `!` indicate an exception rule, allowing to include some files in the context even if they would be excluded by some previous rule for instance

```
*.md
!README.md
```

!README.md is included in the context, the exception rule should be written after the rule that has to be modified in order to have effect.

## FROM

```
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
```

- A valid Dockerfile must start with a `FROM` instruction. the image can be any valid image.
- `ARG` instruction is the only instruction that can preceed FROM
- `FROM` can appear multiple times in a Dockerfile for either create multiple images or use one build stage as a dependency for another one. 

The platform flag let's you specify the pltform the image will run upon in case the image is a multi-platform one. the `:tag` let's you specify the version of the image, if not specified the default value is `:latest` (but you should't use it because everytime you build the image again the version may change and possibly something could get broken), you can also refer to a specific image by its digest using `@digest` instead of `:tag`. You can assign a name to a build stage using `AS name` and therefore refer to that building stage in a following one for instance for copying some files from it in the new one.

`ARG` can be used to define variables, it's possible to define an `ARG` before the first `FROM` which can therefore refer to it.

```
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
```

To use the value of `ARG` declared before the first `FROM` after the `FROM` istruction, you should use an `ARG` instruction without a value into the building stage (sice the first `ARG` would be outside of it).

```
ARG VERSION=latest
FROM busybox:$VERSION
ARG VERSION
RUN echo $VERSION > image_version
```

## RUN

two forms

- shell:

```
RUN <command>
```

- exec:

```
RUN ["executable", "param1", "param2"]
```

I wont' go in detail on the exec mode because I think my tool should use the shell mode.. which also allows to chain commands.

Each run command will run in a new layer on top of the current immage and then it will perform a commit at the end of the execution, therefore command chaining is usually used to reduce the final size of the image. You can continue a `RUN` instruction on a new line in shell mode using `\`. The default shell using the shell form is `/bin/sh -c` while the exec form doesn't actually invoke any shell command.

- In the shell form if you want to use a different shell with respect to the default one `RUN` let's you specify the shell you want but then you have to deal with the shell string munging for instance:
```
RUN /bin/bash -c 'source $HOME/.bashrc'
```
- You can change the desired shell to use in shell mode via the `SHELL` instruction.

Docker caches the output of `RUN` instructions and use them in the next build.

## CMD

There is a shell form for `CMD` but I will use only the exec form. 

- exec form:
```
CMD ["executable","param1","param2"]
```
- exec form (used as default arguments for `ENTRYPOINT`)

```
CMD ["param1","param2"]
```

The main purpose of `CMD` is to provide defaults for an executing container. These default can include the executable to run at container start, or if the executable is omitted the default arguments for the `ENTRYPOINT` instruction. Note that the exec form does not actually invoke any shell, therefore shell processing like environment variable expansion does not take place. To use shell processing is possible to execute a shell directly (use `/bin/sh` as executable for instance, passing the command as an argument). The arguments of `CMD` are in the form of a JSON array therefore double quotes are mandatory.

## LABEL

Offers a way to attach metadata to your image, this metadata can be inspected at any time durng the lifecycle of the resulting image.

```
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```

if you want to use spaces you can use double quotes for enclosing both keys and values, if you want to continue the instruction on the next line it's possible to use the backslash. I would attach some basic informations tho the images the tool will produce following the OCI containers specification

- `org.opencontainers.image.created` – Image creation time.
- `org.opencontainers.image.version` – Version of the main software inside the container (not the image’s version).
- `org.opencontainers.image.title` – A human-readable name for the container

## EXPOSE

```
EXPOSE <port> [<port>/<protocol>...]
```

Let's you specify the ports on which the container is listening, optionally you can also specify the protocols (the default is TCP). How the exposed ports will be mapped on the host ports at runtime is out of the scope of my tool.

## ENV

```
ENV <key>=<value> ...
```

Sets the environment variable `<key>` to the value `value`. This environment variable will be persistent and valid for all the subsequent instructions in the building process and while the container derived from this image will be running. To use values which contains spaces is possible to use quotes (that will be removed automatically). For instance:

```
ENV MY_NAME="John Doe"
```
You can set multiple key-value pairs in the same instruction. The fact that the environment variables are persisted also at runtime could cause unexpected behaviours that may confuse the user of the image. Therefore if an environment variable is not needed at runtime but only used during build time is possible to:

- Set the environment variable in the same `RUN` instruction where the variable is needed, by doing this, considering that every new `RUN` instruction will create a new layer, the variable will be valid only for the current instruction

```
RUN DEBIAN_FRONTEND=noninteractive apt-get update
```

- Use the `ARG` instruction, which is not persisted in the final image

```
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update
```
## ADD

```
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

Copies new files, directories or remote file URLs from `<src>` and adds them to the filesystem of the final image at the path `<dest>`. There is also onther form of this command but I will use this one since it allows to also use whitespaces in the path names. It is possible to specify multiple `<src>` and each one will be considered as a relitive path from the root of the build context, it is also possible to use wildcards for them, following the Go's [filepath.Match](https://pkg.go.dev/path/filepath#Match) (like what we had for the dockerignore syntax <a href="#dockerignore">above</a>). The `<dest>` is an absolute path or a path relative to `WORKDIR`. When writing files or directories names which contains special characters that could be interpreted as matching patterns, they must be escaped according to the Golang rules. The flag `--chown` can be used to specify an ownership of the content added, otherwise the new files will get a UID and GID of 0. Is it possible in the `<user>:<group>` pair to specify a name or direclty an integer in any combination. The conversion between names and UIDs/GIDs will be performed by doing a lookup of the `/etc/passwd` and `/etc/group` files in the container filesystem. If names are used and the container filesystem does not contain either `/etc/passwd` or `/etc/group` files then the build will fail, by using UID and GID directly instead the build process will be indipendent from the container's filesystem content. It's possible to specify only the username/UID without a group, in tht case the GID will be equal to te UID provided (or translated from the username). The default behaviour is to give full permissions to the user specified in the flag (700), unless the the `<src>` is a remote file URL, in that case the execution priviledge is dropped (600). When getting the file from a remote URL if the HTTP response contains a `Last-Modified` header, this will used as `mtime` field for the file in the container, however the `mtime` field will not influence the caching used by Docker during the build phase. If the `<src>` is a remote URL that requires authentication it is not possible to use `ADD` to copy the file in the destination as it does not suopport authentication, `RUN wget` can be used instead or any other tool from within the container. `ADD` instruction obeys the following rules:
- the `<src>` path must refer to a file inside the build context
- if `<src>` is a URL and `<dest>` finishes with a trailing slash than the complete path of the file in the container will be `<dest>/<filename>` where  `<filename>` is inferred from the URL (the URL must contain a non trivial path to the file to be downloaded), otherwise if `<dest>` does not finish with a trailing slash the file is downloaded from the URL and copied in `<dest>`.
- if `<src>` is a directory its entire content is copied.
- if `<src>` is a local archive, it is authomatically decompressed into a folder in the destination, for remote archives from URLs, the decompression is not performed. When copying or unpackaging directories the confilcts are resolved by overriding the previous content in favor of the newly copied one.
- if `<src>` is not an URL, not a directory and not an archive (any other file kind), if `<dest>` ends with a trailing slash the final destination path will be `<dest>/basename(<src>)`.
- if multiple `<src>` are specified by listing them or by using wildcards, then the `<dest>` must be a folder and must end with a slash.
- if `<dest>` does not finish with a trailing slash it will be consiered a regular file and the contents of `<src>` will be written at `<dest>`
- if `<dest>` does not exists it will be created along with all missing subdirectories in its path.

## COPY

Is complitely identical to the `ADD` instruction with the difference that can be used only for copying local files or directories (no remote URLs) or optionally files coming from previous build stages (created with the instruction `FROM .. AS <name>`) by using the flag `--from=<name>`, this last option is very useful for multi-stage builds.

## ENTRYPOINT

```
ENTRYPOINT ["executable", "param1", "param2"]
```

Allows you to specify the executable the container has to execute when runned. The parametes specified in the `ENTRYPOINT` isntruction will always be applied while the arguments specified with the `CMD` instruction will be appended if not overrided by the `docker run <image>` command that can accept argument to pass to the executable of the `ENTRYPOINT`. This for does not perform shell processing because no command shell is invoked (this is the exec mode, I won't use the shell mode).

## VOLUME

```
VOLUME ["/data"]
```

Makes possible to create mountpoints and mark them as holding externally mounted volumes from the host or from other containers. The value can be a JSON array (double quotes are needed) or a string with whitespaces to separate the directories. It's important to notice that after declaring the volume, all the modifications to the volumes derived from the next building steps will be discarded.

## USER

```
USER <user>[:<group>]
```

`<user>` and `<group>` can be both names or integers (`<UID>` and `<GID>`), but they have to be of the same type. Sets the user and optionally the group to be used when running the image and to run all the following `RUN`, `CMD` and `ENTRYPOINT` instructions in the Dockerfile.

## WORKDIR

```
WORKDIR /path/to/workdir
```

Sets the working directory for any following `RUN`, `CMD`, `ENTRYPOINT`, `COPY` and `ADD` instructions. If the `WORKDIR` doesn't exist it will be created. The `WORKDIR` instruction can be used multiple times in the Dockerfile, if a relative path is provided, it will be relative to the path set by the last `WORKDIR` instruction. When building an image from scratch (`FROM scratch`) the default `WORKDIR` is `/`, otherwise it is likely that teh `WORKDIR` has been set to a different path by the base image you'll be use. To avoid unintended operations in unknown directories it is best practice to set the `WORKDIR` explicitily.

## ARG

```
ARG <name>[=<default value>]
```

Defines a variable that can be passed by the user at build time (if the user sets a variable tht is not defined in the Dockerfile, the build outputs a warning). By specifying a default value this will give a value to the variable if the user does not override it at building time. The variable definition comes into effect from the line on which is defined (not before). The `ARG` instruction goes out of scope et the end of the building stage, therefore if the same variable is needed in multiple stages, it must be included in each of them. `ARG` can be used like `ENV` to setup environment variable that are available to `RUN` instructions. Environment variable define with `ENV` with the same name will always override the ones defined with `ARG`. The variables defined with `ARG` are not persisted in the final image. Although `ARG` variables are not persisted in the final image, their declaration might affect the caching mechanism during the build, in particular performing 2 builds using different values for an `ARG` variable may cause a following `RUN` instruction to produce a chache miss (even if it not use the variable).

## ONBUILD

```
ONBUILD <INSTRUCTION>
```

Is used to add to the image trigger instructions that will be executed when the image will be used as a base image for a future build. The instructions will be executed right after the `FROM` instruction in the same order they were registered in the base image. I don't think I will need this instruction.

## STOPSIGNAL

```
STOPSIGNAL signal
```

Is used to set the system call signal to be sent to the container to terminate it. It can be specified in the format `SIG<NAME>` (for instance `SIGKILL`), or as an unsigned number. The default signal is `SIGTERM`.

## HEALTCHECK

Has 2 forms:

- `HEALTHCHECK [OPTIONS] CMD command`
- `HEALTHCHECK NONE`

Is used to provide a command that shoud return a status code that represent the healtiness of a container. The second form is used to disable all the `HEALTHCHECK` that the base image could have set. The second form accepts the following options:

- `--interval=DURATION` (default: `30s`)
- `--timeout=DURATION` (default: `30s`)
- `--start-period=DURATION` (default: `0s`)
- `--retries=N` (default: `3`)

The first check will start after `interval` seconds and then again after `interval` seconds from each previous check that completes. If a single check takes longer than `timeout`, it is considered failed. If a number equal to `retries` of consecutive failures happens, the container is cosidered unhealty. The `start-period` indicates a time interval at the start of the container in which `HEALTHCHECK` does not increment the count of maximum failures registered. There can be only one `HEALTHCHECK` per image, only the last one is valid. The `CMD` can be used in its shell or exec form. The `CMD` instruction should complete with 0 for success and with 1 for failure.

## SHELL

```
SHELL ["executable", "parameters"]
```

Allows to override the default choice for the shell to use for commands in shell form. The default one is `["/bin/sh", "-c"]`. I don't think I will use this instruction as it is mainly useful with windows containers, which I am not planning to consider for the moment.