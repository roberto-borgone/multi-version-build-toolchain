# Python

What is needed is just the python environment with its proper version and the dependencies modules our application is based upon. Let's assume that the user provides us the dependencies creating a `requirement.txt` file (this can be done using `pip freeze > requirements.txt` in the root folder of the Python project). Here's an example of such file.

```
###### Requirements without Version Specifiers ######
nose
nose-cov
beautifulsoup4

###### Requirements with Version Specifiers ######
docopt == 0.6.1             # Version Matching. Must be version 0.6.1
keyring >= 4.1.1            # Minimum version 4.1.1
coverage != 3.5             # Version Exclusion. Anything except version 3.5
Mopidy-Dirble ~= 1.1        # Compatible release. Same as >= 1.1, == 1.*
```
Maybe Gradle is not needed for a Python project, there are ways to use Gradle to extend the things the Python ecosystem can do (PyGradle from Linkedin) but I don't think I will use it.

## Docker
How to package a simple python script in a docker container, there is the official python base image in Docker Hub which already contains the python enviroment.

### Dockerfile

An example of how the Dockerfile of a python application should look like.

```
# use python:2 for Python 2

FROM python:3 

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD [ "python", "./your-daemon-or-script.py" ]
```

## Ansible

We could assume that Python is already installed in the target machines as Ansible modules heavily rely on Python (and also the idempotency mechanism rely on it), so even though it is not mandatory many Ansible installations also include a cluster-wide installation of python.

```

```


