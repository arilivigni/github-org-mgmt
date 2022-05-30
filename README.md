# github-org-mgmt
- [Installing Jenkins](#installing-jenkins)
  * [Prerequisites](#prerequisites)
  * [Disabling the Setup Wizard](#disabling-the-setup-wizard)
  * [Installing Jenkins Plugins](#installing-jenkins-plugins)
  * [Setup JCasC](#setup-JCasC)
    * [Specifying the Jenkins URL](#specifying-the-jenkins-url)
    * [Creating a User](#creating-a-user)
    * [Setting Up Authorization](#setting-up-authorization)
  * [Building and Running the Jenkins Container Image](#building-and-running-the-jenkins-container-image)
    * [Build Jenkins Container Image](#build-jenkins-container-image)
    * [Run Jenkins Container Image](#run-jenkins-container-image)
- [Jenkinsfile - Webhook Trigger and apply branch protection rules](jenkinsfile-webhook-trigger-branch-protection-rules)

# Overview
The purpose of this repository is to manage github organizations and currently focused on branch protection rules.
It also contains the building blocks to have infrastructure as code in setting up the Jenkins server to communicate to Github.
This is accompleshed by deplying Jenkins in a container using Docker plus using the [JCasC](https://www.jenkins.io/projects/jcasc/) (Jenkins Configuration as Code) plugin
so the entire environment can be updated, built, tested, and deployed via Github.

## Installing Jenkins
This project details setting up a Jenkins server with customizations using [JCasC](https://www.jenkins.io/projects/jcasc/) (Jenkins Configuration as Code) using the Configuration as Code plugin.

#### Prerequisites
* Access to a server with at least 2GB of RAM and Docker installed. This can be your local development machine, cloud server, or any kind of server
* Exposing localhost to the internet we're going to use a local Jenkins server to receive messages from GitHub. So, first of all, we need to expose 
  our local development environment to the internet. We'll use ngrok to do this. ngrok is available, free of charge, for all major operating systems. 
  For more information, see [the ngrok download page](https://ngrok.com/download).

### Disabling the Setup Wizard
Using JCasC eliminates the need to show the setup wizard; therefore, you’ll create a modified version of the official `jenkins/jenkins` image 
that has the setup wizard disabled. You will do this by creating a `Dockerfile` and building a custom Jenkins image from it.

The `jenkins/jenkins` image allows you to enable or disable the setup wizard by passing in a system property named `jenkins.install.runSetupWizard`
via the `JAVA_OPTS` environment variable. Users of the image can pass in the JAVA_OPTS environment variable at runtime using the `--env` flag to `docker run`. 
However, this approach would put the onus of disabling the setup wizard on the user of the image. 
Instead, you should disable the setup wizard at build time, so that the setup wizard is disabled by default.

You can achieve this by creating a `Dockerfile` and using the `ENV` instruction to set the `JAVA_OPTS` environment variable.
You’re using the FROM instruction to specify jenkins/jenkins:latest as the base image, and the ENV instruction to set the JAVA_OPTS environment variable.

```
FROM jenkins/jenkins:latest
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
```

### Installing Jenkins Plugins
In this section you are going to modify the `Dockerfile` to pre-install a selection of plugins, including the Configuration as Code plugin.

To automate the plugin installation process, you can make use of an installation script that comes with the `jenkins/jenkins` Docker image. You can find it inside the container at `/usr/local/bin/install-plugins.sh`. To use it, you would need to:

Use `jenkins/plugins.txt` as a guide containing a list of plugins to install
Copy it into the Docker image
Run the `install-plugins.sh` script to install the plugins
First, using your editor, create a new file named `plugins.txt`:

Then, add in the following newline-separated list of plugin names and versions (using the format `<id>:<version>)`:
Example:
```
ant:latest
antisamy-markup-formatter:latest
build-timeout:latest
cloudbees-folder:latest
configuration-as-code:latest
credentials-binding:latest
email-ext:latest
git:latest
github-branch-source:latest
gradle:latest
ldap:latest
mailer:latest
matrix-auth:latest
pam-auth:latest
pipeline-github-lib:latest
pipeline-stage-view:latest
ssh-slaves:latest
timestamper:latest
workflow-aggregator:latest
ws-cleanup:latest
```

Next, open up the Dockerfile file:

In it, add a `COPY` instruction to copy the `plugins.txt` file into the `/usr/share/jenkins/ref/` directory inside the image; this is where Jenkins normally looks for plugins. Then, include an additional `RUN` instruction to run the `install-plugins.sh` script.  Now your Dockerfile looks like below:

```
FROM jenkins/jenkins
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt
```

## Setup JCasC

The below are specific changes we want to make to the Jenkins configuration when we build the Jenkins container image.

### Specifying the Jenkins URL
The Jenkins URL is a URL for the Jenkins instance that is routable from the devices that need to access it. For example, if you’re deploying Jenkins as a node inside a private network, the Jenkins URL may be a private IP address, or a DNS name that is resolvable using a private DNS server. For this tutorial, it is sufficient to use the server’s IP address (or `127.0.0.1` for local hosts) to form the Jenkins URL.

You can set the Jenkins URL on the web interface by navigating to `server_ip:8080/configure` and entering the value in the Jenkins URL field under the Jenkins Location heading. Here’s how to achieve the same using the Configuration as Code plugin:

1. Define the desired configuration of your Jenkins instance inside a declarative configuration file (`jenkins/jcasc.yaml`).
2. Copy the configuration file into the Docker image (just as you did for your `plugins.txt` file).
3. Set the `CASC_JENKINS_CONFIG` environment variable to the path of the configuration file to instruct the Configuration as Code plugin to read it.

First, use the file `jenkins/jcasc.yaml` for your use case or modify as you wish.
This how your `Dockerfile` will look when including JCasC in the container image.

```
FROM jenkins/jenkins:latest
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
ENV CASC_JENKINS_CONFIG /var/jenkins_home/casc.yaml
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt
COPY jcasc.yaml /var/jenkins_home/jcasc.yaml
```

### Creating a User
So far, your setup has not implemented any authentication and authorization mechanisms.Now you will set up a basic, password-based authentication scheme and create a new user named admin.

Start by opening your jcasc.yaml file:

Then, add in the highlighted snippet:

```
jenkins:
  securityRealm:
    local:
      allowsSignup: false
      users:
       - id: ${JENKINS_ADMIN_ID}
         password: ${JENKINS_ADMIN_PASSWORD}
unclassified:
  ...
```
In the context of Jenkins, a security realm is simply an authentication mechanism; the local security realm means to use basic authentication where users must specify their ID/username and password. Other security realms exist and are provided by plugins. For instance, the LDAP plugin allows you to use an existing [LDAP](https://plugins.jenkins.io/ldap) directory service as the authentication mechanism. The [GitHub Authentication](https://plugins.jenkins.io/github-oauth) plugin allows you to use your GitHub credentials to authenticate via OAuth.

Note that you’ve also specified allowsSignup: false, which prevents anonymous users from creating an account through the web interface.

Finally, instead of hard-coding the user ID and password, you are using variables whose values can be filled in at runtime. This is important because one of the benefits of using JCasC is that the `jcasc.yaml` file can be committed into source control; if you were to store user passwords in plaintext inside the configuration file, you would have effectively compromised the credentials. Instead, variables are defined using the `${VARIABLE_NAME}` syntax, and its value can be filled in using an environment variable of the same name, or a file of the same name that’s placed inside the `/run/secrets/` directory within the container image.

### Setting up Authorization
After setting up the security realm, you must now configure the authorization strategy. In this step, you will use the [Matrix Authorization Strategy](https://plugins.jenkins.io/matrix-auth) plugin to configure permissions for your admin user.

By default, the Jenkins core installation provides us with three authorization strategies:

* `unsecured:` every user, including anonymous users, have full permissions to do everything
* `legacy:` emulates legacy Jenkins (prior to v1.164), where any users with the role admin is given full permissions, whilst other users, including anonymous users, are given read access.

All of these authorization strategies are very crude, and does not afford granular control over how permissions are set for different users. Instead, you can use the Matrix Authorization Strategy plugin that was already included in your `plugins.txt` list. This plugin affords you a more granular authorization strategy, and allows you to set user permissions globally, as well as per project/job.

The Matrix Authorization Strategy plugin allows you to use the `jenkins.authorizationStrategy.globalMatrix.permissions` JCasC property to set global permissions.

```
...
       - id: ${JENKINS_ADMIN_ID}
         password: ${JENKINS_ADMIN_PASSWORD}
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"
unclassified:
...
```
The `globalMatrix` property sets global permissions (as opposed to per-project permissions). The `permissions` property is a list of strings with the format `<permission-group>/<permission-name>:<role>`. Here, you are granting the `Overall/Administer` permissions to the `admin` user. You’re also granting `Overall/Read` permissions to `authenticated`, which is a special role that represents all authenticated users. There’s another special role called `anonymous`, which groups all non-authenticated users together. But since permissions are denied by default, if you don’t want to give anonymous users any permissions, you don’t need to explicitly include an entry for it.

## Building and Running the Jenkins Container Image

At this point we have the plugins and Jenkins configuration we want so we can build and run our Jenkins container image.

### Build Jenkins Container Image
```
docker build -t jenkins:jcasc .
```

### Run Jenkins Container Image
```
docker run --name jenkins --rm -p 8080:8080 --env JENKINS_ADMIN_ID=admin --env JENKINS_ADMIN_PASSWORD=password jenkins:jcasc
```

You can open a browser and go to `http://server_ip:8080` to login and start using Jenkins

## Jenkinsfile - Webhook Trigger and apply branch protection rules

### Defining our actions
We'll be defining our _payload_ to execute the following actions against the **GitHub REST API**.

- Protect the `main` branch, which may or may not be named _main_. In this demo, it is named _main_
- `"required_approving_review_count": 2`
- `"strict": true`
- ```
  "contexts": [
      "continuous-integration/jenkins/branch"
    ]
  ```
The payload is defined in `JSON` format and will be stored in the pipeline as a _variable_

### Defining the payload

```json
{
  "required_status_checks": {
    "required_approving_review_count": 2,
    "strict": true,
    "contexts": [
        "continuous-integration/jenkins/branch"
    ]
  }
```
