# Create a simple pipeline using CodeCommit repository

[![AWS-Deploying_and_Managing_Infrastructure](https://img.youtube.com/vi/UzMdlSA1X1Q/0.jpg)](https://www.youtube.com/watch?v=UzMdlSA1X1Q)

---

As a follow up to the presentation, I would like to present a demo of a simple pipeline using CodeCommit repository.

> We are going to use `us-east-1` region for this demo.


![Pipeline-Architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rjg9dx4g9ayqc08k81uf.png)

## Table of Contents
1. [Create AWS CLI User](#cliuser)
2. [Create IAM Roles](#iam)
3. [Create a CodeCommit repository](#codecommit)
4. [Add sample code to your CodeCommit repository](#code)
5. [Create an EC2 Linux instance and install the CodeDeploy agent](#ec2)
6. [Create an application in CodeDeploy](#codedeploy)
7. [Create a pipeline in CodePipeline](#pipeline)
8. [Modify code in your CodeCommit repository](#modify)
9. [Clean Up](#cleanup)
10. [Summary](#summary)
11. [Referrals](#referrals)

---

<a name="cliuser"></a>

### Step 1. Create AWS CLI User

1. Navigate to `IAM` > `Users` > Click `Add users`
2. Enter a `User Name`, Select `Access key - Programmatic access` and Click `Next: Permissions`
3. Select `Administator` at `Add user to group` and Click `Next: Tags`
4. Click `Next: Review`
5. Click `Create user`
6. Click `Download.csv` and `Close`
7. Again navigate to `IAM` > `Users` > Click on the user that was created in the previous step
8. Navigate to `Security Credentials`
9. Click on `Generate Credentials` under `HTTPS Git credentials for AWS CodeCommit` 
10. Click `Download credentials` and `Close`
11. Setup AWS CLI as shown below. 


![IAM1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kwju7ih2cubmh208656z.png)


![IAM2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/63wthhh6mv9ckv94ocs0.png)


![IAM3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/akzucbqdmcaxcwbwvb53.png)


![IAM4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3lbjg03vhu94ndud7xub.png)


![IAM5](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/irp7gwcqlwme4vvurlwu.png)


![aws-configure](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fqv527uwglo1iuskn1hh.jpg)

---

<a name="iam"></a>

### Step 2. Create IAM Roles

We are going to create two IAM Roles: one for EC2 and the other one for Code Deploy

**EC2InstanceRole**

1. Navigate to `IAM` > click `Roles` >  click `Create role`.
2. Select `EC2` under `Common use cases` and click `Next`.
3. Search for and select the policy named `AmazonEC2RoleforAWSCodeDeploy`, and then click `Next`.
4. Enter a name for the role as `EC2InstanceRole` and then click `Create role`.


![EC2InstanceRole](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cat5xy3ktm89gi5w7h2o.png)

**CodeDeployRole**

1. Navigate to `IAM` > click `Roles` >  click `Create role`.
2. Under `Use cases for other AWS services:` > Select `CodeDeploy` and click `Next`.
3. `AWSCodeDeployRole` managed policy will be automatically added, so click `Next`
4. Enter a name for the role as `CodeDeployRole` and then click `Create role`.


![CodeDeployRole](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4n5372vatas1yrtaut0u.png)
---

<a name="codecommit"></a>

### Step 3. Create a CodeCommit repository

1. Navigate to `CodeCommit`
2. On the `Repositories` page >  click `Create repository` > Enter `MyDemoRepo` as Repository name and then click `Create`.
3. Copy the `git clone` command from `Clone the repository` and execute the following command

```bash
mkdir codedeploy
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyDemoRepo
Cloning into 'MyDemoRepo'...
warning: You appear to have cloned an empty repository.
```
> Note: Enter `git credentials` when cloning the repo for the first time.


![create-repo](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vz9qiuuaupc3tut0t3de.png)

---

<a name="code"></a>

### Step 4. Add sample code to your CodeCommit repository

1. We will clone github code to `/tmp` folder and then copy the contents to codecommit's `MyDemoRepo`

    ```bash
    mkdir -p /tmp;
    cd /tmp;
    git clone https://github.com/kasukur/codedeploy.git
    cp -rf codedeploy/MyDemoRepo/* ~/codedeploy/MyDemoRepo/.
    ```

2. Commit and push the files to CodeCommit `MyDemoRepo` repository

    ```
    cd MyDemoRepo
    git add -A
    git commit -m "Add sample application files"
    git push
    ```

> Note: I have used the code from [SampleApp_Linux.zip](https://docs.aws.amazon.com/codepipeline/latest/userguide/samples/SampleApp_Linux.zip) and [Solid State HTML5 template](https://html5up.net/solid-state). Alternatively, you can download both manually and add them to codecommit's `MyDemoRepo`

The repository tree should look like this:

```
MyDemoRepo
       │-- appspec.yml
       │-- index.html
       │-- LICENSE.txt
       └-- scripts
           │-- install_dependencies
           │-- start_server
           └-- stop_server
```           

AppSpec file must be a YAML-formatted file named appspec.yml and it must be placed in the root of the directory structure of an application's source code, otherwise deployments fail. It is used by CodeDeploy to determine:
    - What it should install onto your instances from your application revision in Amazon S3 or GitHub.
    - Which lifecycle event hooks to run in response to deployment lifecycle events.

```yaml
version: 0.0
os: linux
files:
  - source: /index.html
    destination: /var/www/html/
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies
      timeout: 300
      runas: root
    - location: scripts/start_server
      timeout: 300
      runas: root
  ApplicationStop:
    - location: scripts/stop_server
      timeout: 300
      runas: root
```


![codecommit-repository](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s99efdwy8qowhxpauw5a.png)

---

<a name="ec2"></a>

### Step 5. Create an EC2 Linux instance and install the CodeDeploy agent

We are going to create an EC2 instance, where we deploy a sample application using CodeDeploy agent.
We are also going to attach an IAM role `EC2InstanceRole` to the instance (known as an instance role) to allow it to fetch files that the CodeDeploy agent uses to deploy your application.

1. Navigate to `EC2` > `Instances` > Click `Launch Intances`
2. Enter Name as `MyCodePipelineDemo`
3. Select `Amazon Linux 2 AMI1` and Instance Type `t2.micro Free tier eligible`
4. You could create a key pair or choose Proceed without a key pair. Creating a key pair would allow you to logon to the EC2 instance and check the application.
5. Select `Allow SSH traffic from My IP` and `Allow HTTP traffic from the internet`
6. Click `Advanced Details` > Select `EC2InstanceRole` under `IAM instance profile` and enter the following in the `user data` and click `Launch Instance`

> Note: New EC2 console adds a tag with `key: Name` and `Value: MyCodePipelineDemo`. In case you don't see the Tags, please add them under `Tags` section by selecting the EC2 instance in EC2 console.

```bash
#!/bin/bash
sudo yum -y update
sudo yum install -y ruby
sudo yum install -y aws-cli
sudo cd /home/ec2-user
sudo wget https://aws-codedeploy-us-east-2.s3.us-east-2.amazonaws.com/latest/install
sudo chmod +x ./install
sudo ./install auto
```

> Note: the user data installs ruby, aws-cli and CodeDeploy agent


![EC2-1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lof7yotbidk65ketdpuk.png)


![EC2-2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/87yy2bpojtn30eapg6qf.png)


![EC2-3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/eq98axdow02iqmn2e3iw.png)


![EC2-4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/el0lcvlygptwr4z4vu1b.png)
---

<a name="codedeploy"></a>

### Step 6. Create an application in CodeDeploy

**Create an application in CodeDeploy**
1. Navigate to `CodeDeploy` > `Applications`.
2. In Application name, enter `MyDemoApplication`.
3. In Compute Platform, choose `EC2/On-premises`.
4. Choose `Create application`.

**Create a deployment group in CodeDeploy**
A deployment group is a resource that defines deployment-related settings like which instances to deploy to and how fast to deploy them.

1. On the page that displays your application, choose `Create deployment group`.
2. In Deployment group name, enter `MyDemoDeploymentGroup`.
3. In Service Role, choose the service role `CodeDeployRole`, which we created earlier.
4. Under `Deployment type`, choose `In-place`.
5. Under `Environment configuration`, choose `Amazon EC2 Instances`. In the Key field, enter `Name`. In the Value field, enter the `MyCodePipelineDemo`.
6. Under `Deployment settings`, choose `CodeDeployDefault.OneAtaTime`.
7. Under `Load Balancer`, make sure `Enable load balancing` is not selected. You do not need to set up a load balancer or choose a target group for this example.
8. Click `Create deployment group`.


![CodeDeploy1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/15deyr0rn40jot8nqsao.png)


![CodeDeploy2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/040gm60jx01g02cw7s6v.png)


![MyDemoApplication](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k6v3xesyuosph6wvsqpo.png)

---

<a name="pipeline"></a>

### Step 7. Create a pipeline in CodePipeline

We are now ready to create the pipeline. 
In this step, we create a pipeline that runs automatically when code is pushed to your CodeCommit repository.

**Create a CodePipeline pipeline**

1. Navigate to `Pipelines` > click `Create pipeline`.
2. In Pipeline name, enter `MyFirstPipeline`.
3. In Service role, choose `New service role` to allow CodePipeline to create a service role in IAM and click `Next`
4. Select `AWS CodeCommit` in Source provider. Select `MyDemoRepo` in Repository name. Select `master` in Branch name.
   Select `Amazon CloudWatch Events (recommended)` under `Change detection options`. Select `CodePipeline default` under `Output artifact format` and click `Next` .
5. Click `Skip build stage` as we are deploying a static website.
6. Click `skip` at `Your pipeline will not include a build stage. Are you sure you want to skip this stage?`
7. At deploy stage,  Select `AWS CodeDeploy` in `Deploy provider`, Select `MyDemoApplication` in `Application name`,  Select `MyDemoDeploymentGroup` in `Deployment group` and then click `Next` .
8. Review the information and then click `Create pipeline`.
9. The pipeline starts running after it is created. It downloads the code from your CodeCommit repository and creates a CodeDeploy deployment to your EC2 instance. You can view progress and success and failure messages as the CodePipeline sample deploys the webpage to the Amazon EC2 instance in the CodeDeploy deployment.

    > If Deploy fails with an error, one of the reasons could be that `codedeploy-agent` may not be running on the EC2 instance, which we can check by running the following command

    ```
    sudo service codedeploy-agent status
    The AWS CodeDeploy agent is running as PID 3436
    ```

10. Now the pipeline has succeeded but when we check `index.html`. it appears that the stylesheets...etc are missing
Let's fix this by updating `appspec.yml`
from: `source: /index.html` to `source: /`

    ```yaml
    version: 0.0
    os: linux
    files:
    - source: /
        destination: /var/www/html/
    hooks:
    BeforeInstall:
        - location: scripts/install_dependencies
        timeout: 300
        runas: root
        - location: scripts/start_server
        timeout: 300
        runas: root
    ApplicationStop:
        - location: scripts/stop_server
        timeout: 300
        runas: root
    ```

    > If source is a single slash ("/" for Amazon Linux, RHEL, and Ubuntu Server instances, or "\" for Windows Server instances), then all of the files from your revision are copied to the instance.

11. Commit and push the changes
    ```
    git add -A
    git commit -m "updated appspec.yml"
    git push
    ```

12. The pipeline will automatically pick up the changes and run the Deploy because of `Amazon EventBridge` Rule


![pipeline1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wx0qwnshpk2ard7remvc.png)


![pipeline2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hxuthdac2glh9fi5t8es.png)


![pipeline3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/grp98hpzqhbch79xzj1d.png)


![pipeline4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/01ueyurw5e8uk447q4sy.png)


![deploy-result](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lg4lcvnkp245by937mpz.png)


![missing-css](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1nvdxmr897jiz71in35g.png)
> Step 9

![website](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/edubzx9rc6noeo33c307.png)
> Step 10

![EventBridge-Rule](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/65xx2p4b3wquqc1heoo6.png)
> Step 12

---

<a name="modify"></a>

### Step 8. Modify code in your CodeCommit repository

1. Update the following line of HTML code in `index.html` and push the changes.
   Change the value from: `This is Solid State`, to: `This is Solid State - Version 1`
2. Commit and push the changes

    ```
    git status
    git commit -am "updated index.html"
    git push
    ```

3. index.html got updated on our EC2 instance.
    ```
    grep -i version /var/www/html/index.html 
    <h2>This is Solid State - Version 1</h2>
    ```                    

![website-final](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d5faqyvcb7lbi64ib0su.png)

---

<a name="cleanup"></a>

### Step 9. Clean Up

1. Terminate `MyCodePipelineDemo` EC2 Instance.
2. Delete `MyFirstPipeline` under `Pipelines`, which will remove `Amazon CloudWatch Events rule` related to the pipeline.
3. Delete `MyDemoApplication` under `Applications`.
4. Delete `MyDemoRepo` under `CodeCommit` > `Repositories`.
5. Empty and delete S3 bucket prefixed with codepipeline.

---

<a name="summary"></a>
### Summary
- In this demo, we learnt how easy it is to setup a simple pipeline using CodeCommit, CodeDeploy and Pipeline.

---

<a name="referrals"></a>
### Referrals
- [Tutorial: Create a simple pipeline (CodeCommit repository)](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-simple-codecommit.html)
- Cover Image by [@darya_jumelya](https://unsplash.com/@darya_jumelya) from [unsplash](https://unsplash.com/photos/eiANX2OqRFM).

---