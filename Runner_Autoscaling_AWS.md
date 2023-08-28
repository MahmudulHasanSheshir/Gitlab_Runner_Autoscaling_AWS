# GitLab Runner Autoscaling Using AWS EC2 Instances

## Overview:
This documentation provides a comprehensive guide on implementing GitLab Runner autoscaling using Amazon Web Services (AWS) Elastic Compute Cloud (EC2) instances. GitLab Runner autoscaling optimizes resource utilization and reduces costs by dynamically provisioning and terminating EC2 instances based on the workload. This guide covers the setup, configuration, and best practices for achieving efficient and flexible GitLab CI/CD pipelines through autoscaling with AWS EC2 instances.


## Introduction

GitLab Runner, a key component of GitLab's CI/CD toolset, executes tasks defined in `.gitlab-ci.yml` files, streamlining development workflows. However, as project demands vary, static resources can lead to inefficiencies.GitLab Runner autoscaling using Amazon Web Services (AWS) Elastic Compute Cloud (EC2) instances solves this challenge. By dynamically provisioning and terminating EC2 instances based on workload, GitLab Runner can adapt to varying demands efficiently.

### Advantages

- **Optimized Costs:** Autoscaling ensures resources are allocated only when needed, reducing expenses.
- **Flexibility:** Resources automatically adjust to demand, maintaining performance during peaks.
- **Scalability:** The system grows or shrinks in response to changes, maintaining responsiveness.
- **Resource Utilization:** Efficiently utilizes resources to avoid waste.
- **Swift Deployment:** Quick provisioning of instances accelerates deployment times.

## Prerequisites

To implement GitLab Runner autoscaling using AWS EC2, you'll need:

- A GitLab account for CI/CD management.
- An AWS account to leverage EC2 resources.
- An AWS IAM user with necessary permissions.

## Private GitLab server setup

For the project, a private GitLab server has been established. The GitLab server has been provisioned in an Ubuntu machine and for installation, GitLab official installation [documentation](https://docs.gitlab.com/omnibus/installation/) is followed.

![](gitlab-dashboard.PNG "GitLab Project Dashboard")

## AWS IAM user configuration

To spin up new runner instances in AWS an IAM user access is needed to the GitLab server. In order to do that an IAM user is created in AWS IAM service with the name "user-gitlab". 
![](IAM.PNG)
After creating the IAM user now we have to give the required access permissions to the user in order to create EC2 instances in AWS console. As AWS has already created some policies for the ease of work, we'll choose the necessary policies from the "Attach necessary policies"
![](IAM1.PNG)
To perform the autoscaling and caching operation full access to AWS EC2 and S3 bucket is essential.
![](IAM2.PNG)

After providing the required access to the IAM user now we need to specify the user credentials for secure operations. In order to create user access credentials we have to follow the given instructions:
IAM > User > USer_name > Security_Credentials > Create_Access_Key
![](IAM3.PNG)
As we will be only using this user account to create and terminate instances we don't have to give it console access. Accessing the AWS command line interface would be perfect for the user.
![](IAM4.PNG)
Now, access credentials are generated and provided in a `.csv` file, we have to store it securely for future.
![](IAM5.PNG)

## Setting Up GitLab Runner Autoscaling

Configuring GitLab Runner to enable autoscaling involves several steps:

### Step 1: Install and Configure GitLab Runner

GitLab runner is installed using the official documentation of GitLab. [Click here](https://docs.gitlab.com/runner/install/linux-repository.html) for the installation guide. To meet the requirement of autoscaling runner we have to install another package named "GitLab Multi-runner". After installing the runner the below command is executed to set the path of the package to source.

```
cat <<EOF | sudo tee /etc/apt/preferences.d/pin-gitlab-runner.pref
Explanation: Prefer GitLab provided packages over the Debian native ones
Package: gitlab-runner
Pin: origin packages.gitlab.com
Pin-Priority: 1001
EOF
```
After souring the path the machine is updated to install the latest package of gitlab multi-runner.
```
sudo apt-get update
sudo apt-get install gitlab-runner
```
Now download Docker machine in the runner machine which will allow the runner to install docker and necessary tooks to create docker environment in target machines. [Click here](https://docs.docker.com/machine/install-machine/) to install docker machine.

For environment setup 

### Step 2: Choose AWS EC2 as the Executor

To register the GitLab runner to the project we used the generated registration token by the gitlab server. In our GitLab Runner configuration file `/etc/gitlab/config.toml`, the executor is set to `docker+machine` to enable autoscaling using Docker Machine.

### Step 3: Define Autoscaling Strategy

The autoscaling strategy is configured based on our project's requirements. We set thresholds for scaling up or down and specified the instance types to use.

By following these steps, we have configured GitLab Runner autoscaling with AWS EC2 instances up and running, ready to optimize our CI/CD pipelines dynamically.

