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
Now download Docker machine in the runner machine which will allow the runner to install docker and necessary tooks to create docker environment in target machines. [Click here](https://docs.docker.com/machine/install-machine/) to install docker machine. Lastly setup [docker](https://docs.docker.com/engine/install/ubuntu/) to the runner machine.

### Step 2: Choose AWS EC2 as the Executor

To register the GitLab runner to the project we used the generated registration token by the gitlab server.
`Gitlab project > Settings(Left side navigation) > CICD > RUNNERS > New Project Runner.`

Clicking on the `New Project Runner` will prompt a new window with instructions given how to register a runner to the project along with the registration token.
![](gitlab_runner_registration.PNG)

Following the given instructions  we will be able to register a runner. However, while specifying the executor we have selected `docker+machine`. Because it will work as a docker machine which will handle all the containerization operations in the target servers. 

### Step 3: Define Autoscaling Strategy
In this section we will be discussing how the runner is configured to execute the auto scaling operation. 
The autoscaling strategy is configured based on our project's requirements. We set thresholds for scaling up or down and specified the instance types to use.

By following these steps, we have configured GitLab Runner autoscaling with AWS EC2 instances up and running, ready to optimize our CI/CD pipelines dynamically.
At first, we opened the gitlab runner configuration file which is located at `/etc/gitlab-runner/config.toml`

** Global Configuration **
```
concurrent = 1
check_interval = 1
```
It is said to be the global section of the runner configuration. 
`concurrent`: It defines the limit of the jobs that can one runner can run concurrently or parallely. In our case we have set the limit to 1 so that one runner instance can run only 1 job concurrently.
`check_interval`: It sets the time interval of the runner to check for new job in the gitlab server. We have set the interval of checking for job to 1 sec.

** Runner Configuration **
```
[[runners]]
  name = "docker-machine"
  limit = 10
  url = "https://gitlab.com/"
  id = 27196664
  token = "*******"
  token_obtained_at = 2023-08-24T04:59:29Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker+machine"
```
It is the most crucial part of the configuration where the executor is defined 
`name = docker-machine`: We set the name of the runner to 'docker-machine'
`limit = 10`: We set the max limit of the runner machines to spinup based on the maximum workload.
`url = "https://gitlab.com/"`: It is the base url of the gitlab server that we are using for our project.
`id = 271545`: It is the id of the runner which is auto generated.
`token`: It is the gitlab runner registratoin token that is generated in the server console and discussed in the previous step(2).
`executor = docker+machine`: It is the executor of the runner which also discussed in the previous section.

** `runners.docker` configuration **
```
 [runners.cache]
    Type = "s3"
    Shared = true
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      AccessKey = "AKIAYMSEIBNVDGVTIO5P"
      SecretKey = "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"
      BucketName = "sheshir001"
      BucketLocation = "us-east-1"
```
This configuration is essential for the GitLab Runner to know where and how to store and retrieve cache data. Using S3 for caching can be beneficial for distributed systems where runners may not have access to a local filesystem, or for persisting cache data across runner restarts
`[runners.cache]`: This line signifies the start of the configuration for the runner's cache settings.
`Type = "s3"`: This line sets the type of cache to S3. This means that the runner will use an S3-compatible storage for caching.
`Shared = true`: This line enables the cache to be shared among all runners. This is useful when different runners need to access the same cached data
`MaxUploadedArchiveSize = 0`: This line sets the maximum size of uploaded archives to 0, which means there is no limit
`[runners.cache.s3]`: This line signifies the start of the configuration for the S3 settings
`ServerAddress = "s3.amazonaws.com"`: This line sets the server address of the S3 bucket to "s3.amazonaws.com"
`AccessKey = "AKIAYMSEIBNVDGVTIO5P"`: This line sets the access key for the S3 bucket
`SecretKey = "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"`: This line sets the secret key for the S3 bucket 
`BucketName = "sheshir001"`: This line sets the name of the S3 bucket to "sheshir001"
`BucketLocation = "us-east-1"`: This line sets the location of the S3 bucket to "us-east-1"

** [runners.machine] configuration **
```
 [runners.machine]
    IdleCount = 2
    IdleScaleFactor = 1.2
    IdleCountMin = 0
    MaxBuilds = 10
    MachineDriver = "amazonec2"
    MachineName = "runner-%s"
    MachineOptions = [
"amazonec2-access-key=AKIAYMSEIBNVDGVTIO5P",
"amazonec2-secret-key=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",
"amazonec2-region=us-east-1",
"amazonec2-vpc-id=vpc-03780aad03ecead68",
"amazonec2-subnet-id=subnet-0d2a8dd3dc997ebb0",
"amazonec2-zone=a",
"amazonec2-use-private-address=false",
"amazonec2-tags=runner-manager-name,gitlab-aws-autoscaler,gitlab,true,gitlab-runner-autoscale,true",
"amazonec2-security-group=sheshir-security",
"amazonec2-instance-type=t2.small"]
```
This configuration is important for defining how the GitLab Runner manages machines for running jobs. The settings can be adjusted based on the specific requirements of the tasks that the runner will be performing
