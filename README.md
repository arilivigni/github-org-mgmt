# github-org-mgmt
- [Installing Jenkins](#installing-jenkins)
  * [Prerequisites](#prerequisites)
  * [Disabling the Setup Wizard](#disabling-the-setup-wizard)
  * [Installing Jenkins Plugins](#installing-jenkins-plugins)
  * [Specifying the Jenkins URL](#specifying-the-jenkins-url)
  * [Creating a User](#creating-a-user)
  * [Setting Up Authorization](#setting-up-authorization)
- [Jenkinsfile - Webhook Trigger and apply branch protection rules](jenkinsfile-webhook-trigger-branch-protection-rules)
# Overview
The purpose of this repository is to manage github organizations and currently focused on branch protection rules

## Installing Jenkins
This project details setting up a Jenkins server with customizations using JCasC (Jenkins Configueation as Code)

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

### Installing Jenkins Plugins

### Specifying the Jenkins URL

### Creating a User

### Setting up Authorization

## Jenkinsfile - Webhook Trigger and apply branch protection rules

#### Defining our actions
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

#### Defining the payload

#### Defining the payload

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
