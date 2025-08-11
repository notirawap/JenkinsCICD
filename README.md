# Continuous Integration Flow

## Build Git Webhooks Triggers

**Jenkins job triggers** automate the execution of jobs without manual intervention. Git Webhooks:** Automatically triggers a job when a commit is pushed to a Git repository  or any event happens on Github repository. The repository sends a JSON payload to Jenkins.

First, add a simple Jenkinsfile and push the commit.

<img width="1014" height="683" alt="Image" src="https://github.com/user-attachments/assets/653b51b1-af61-402e-860d-7ac995d3a275" />

<aside>
🚨 Before creating a Jenkins job, it’s important to address the potential **"Host key verification failed"** error. This occurs when Jenkins tries to access the Git repository via SSH for the first time. Like any initial SSH connection to a Linux machine, Jenkins would need to accept the server's fingerprint. If strict host key checking is enabled, the connection will be rejected, resulting in this error. Taking precautionary measures can help prevent this issue. Go to `Manage Jenkins` > `Security` > `Git Host Key Verification Configuration` and select `Accept first connection` option.
</aside>

In Jenkins server, create a job in Jenkins that fetches the `Jenkinsfile` from a repository and runs it. Go to  `Configure`  > `Pipeline` and select `Pipeline script for SCM`. Select `SCM` as `Git`, and `Repository URL` as Git repository SSH URL from Github. To add credentials, select `SSH Username with private key`, name the credentials `ID`, give `Username` as Github account, and fill the private key directly from `cat ~/.ssh/<privateKeyFile>`.  Select Branch and give Script Path as Jenkinsfile path in the repository.

<img width="730" height="857" alt="Image" src="https://github.com/user-attachments/assets/90eab036-3508-4658-847c-95e20b6235db" />
<img width="545" height="547" alt="Image" src="https://github.com/user-attachments/assets/b0510545-becd-4158-a096-e61e7a4883e1" />

Go to `<repositoryName>` > `Settings` > `Webhooks` > `Add webhook`, fill `Payload URL` as `http://<JenkinsServerIP>:8080/github-webhook/`  and `Content type` as `application/json` , and select an event on Github to trigger the webhook.

<aside>
🚨

Make sure port 8080 of Jenkins server should be allowed from any IP.

</aside>

<img width="1200" height="1218" alt="Image" src="https://github.com/user-attachments/assets/fdd4379e-da52-41af-aa37-c833614cfdc1" />

In Jenkins server, go to `Configure` > `Build Triggers` and select `GitHub hook trigger for GITScm polling`.

<img width="742" height="538" alt="Image" src="https://github.com/user-attachments/assets/59fd9a05-3b22-4691-9b16-120b148e8cd6" />

Whenever you push the commit, the job in Jenkins is triggered.

<img width="1022" height="576" alt="Image" src="https://github.com/user-attachments/assets/667b54c7-9c21-41ff-9f3e-bf2f197ccc40" />
<img width="610" height="334" alt="Image" src="https://github.com/user-attachments/assets/ace03f8e-0c1d-4f0d-a4a1-4623f420e664" />


## Fetching a Code by Git

The **Fetch Code** stage in a Jenkins pipeline is responsible for retrieving the project's source code from a version control system.

In `Manage Jenkins` > `Plugins` of Jenkins Dashboard, install `Pipeline Utility Steps` and `Git plugin` plugins. Update this pipeline code in `Jenkinsfile` and build by `Build Now`.

```groovy
stage('Fetch code') {
    steps{
        git branch: 'main', 
        url: 'https://github.com/<username>/dummyApp-project.git'
    }
}
```

The `git` command is part of the Jenkins [**Git plugin**](https://www.jenkins.io/doc/pipeline/steps/git/), which allows Jenkins to interact with Git repositories. **`branch: '<branchName>'`  s**pecifies the branch to be checked out from the Git repository. **`url: '<githubReposURL>'`  d**efines the URL of the Git repository from which Jenkins will fetch the code. Jenkins, using the Git plugin, will connect to the specified Git repository and check out the branch. The code from that branch will be downloaded into the Jenkins workspace for subsequent pipeline stages to process.

## Building and Unit Testing by Maven

The **Build** stage is responsible for compiling the source code, resolving dependencies, and packaging the application into a deployable format (e.g., `.war` or `.jar`). 

The **Unit Test** stage executes automated test cases to validate the functionality of individual components in the codebase.

In `Manage Jenkins` > `Plugins` of Jenkins Dashboard, install `Pipeline Maven Integration` plugin. Then, install `Maven` tools in `Manage Jenkins` > `Tools` > `Maven Installations`.

<img width="2154" height="885" alt="Image" src="https://github.com/user-attachments/assets/44005b35-b4c0-4325-9e3f-cd8097f6f1ac" />

To install Java tools manually on the Linux system, use `apt install openjdk-17-jdk`. After installation, when installing JDK, you must set the `JAVA_HOME` environment variable to the Java installation directory `/usr/lib/jvm/java-17-openjdk-arm64`.

<img width="2145" height="744" alt="Image" src="https://github.com/user-attachments/assets/505b94f5-df6e-43ca-b5fa-2e4cb03d8246" />

Update this pipeline code in `Jenkinsfile` and `Build Now`.

```groovy
stage('Build') {
    steps{
        sh 'mvn install -DskipTests'
    }
    post {
        success{
            echo "Archiving artifact"
            archiveArtifacts artifacts: '**/*.war' 
        }
    }
}

stage('Unit Test') {
    steps{
        sh 'mvn test'
    }
}
```

Both stages rely on Maven, which is integrated with Jenkins using the **Maven Plugin**. The Maven plugin ensures compatibility between Jenkins and Maven, allowing you to use Maven commands like `mvn install` and `mvn test` seamlessly.

1. Build Stage
    - **`mvn install -DskipTests`**: Executes Maven's **lifecycle phases** (compiling the code, processing resources, packaging, and installing the artifact in the local Maven repository); and instructs Maven to skip the execution of test cases during the build process, focusing only on building the artifact.
    - This step generates a `.war` (Web Application Archive) file, typically found in the `target` directory.
    - **`archiveArtifacts artifacts: '**/*.war'` s**aves the `.war` file as an artifact in the Jenkins job. This ensures the artifact is preserved and can be accessed later, even if the workspace is cleaned up.
2. Unit Test Stage
    - Runs the **test phase** of the Maven lifecycle, which executes the unit tests defined in the project. When you run `sh 'mvn test'` in the `Unit Test` stage of your Jenkins pipeline, the test results are saved in the `target/surefire-reports` directory by default.

## Code Analysis by Checkstyle and SonarQube scanner

In security group of **SonarQube server** the inbound rule, port 80 is allowed from Jenkins Security Group that allows SonarQube security group to communicate with Jenkins.

<img width="1359" height="242" alt="Image" src="https://github.com/user-attachments/assets/53405f6f-17d0-48a9-815e-e7bf735403c9" />

In `Manage Jenkins` > `Plugins` of Jenkins dashboard, install `SonarQube Scanner` plugins. Then, install SonarQube Scanner tools in `Manage Jenkins` > `Tools` > `SonarQube Scanner Installations`.

<img width="1571" height="936" alt="Image" src="https://github.com/user-attachments/assets/f3077994-17cb-4364-a02b-5e2d184f0cbe" />

Since we need to upload the result to SonarQube server to scan the code, we need to store the Sonarqube server detail in the Jenkins server in `Manage Jenkins` > `System` > `SonarQube servers`. You give the detail including `Name` as SonarQube server environment name, `Server URL` as `http://<sonarqube private IP>:80`, and `Sever authentication token`. Copy the token generated from `My Account` > `Security` in SonarQube server. In Jenkins server, paste token from SonarQube server in `Secret` and name `ID` in `Sever authentication token` as a `Secret text`.

<aside>
    💡

We will use the private IP of the SonarQube server since Jenkins, SonarQube, and Nexus are in the same VPC and can securely communicate over the internal network. Using a public IP would require opening port 80 in the security group, exposing the server to the internet. By using the private IP, we ensure secure, internal communication within the VPC.

</aside>

<aside>
    💡

We have an NGINX service running on the SonarQube server, listening on port 80, which redirects requests to the SonarQube service on port 9000. This means we don't need to specify the SonarQube port number directly in `Server URL`, as NGINX handles the redirection.

</aside>

<img width="2122" height="1080" alt="Image" src="https://github.com/user-attachments/assets/55f86d6e-9465-444a-8ed5-e4b0a0763169" />

<img width="1549" height="681" alt="Image" src="https://github.com/user-attachments/assets/12d43bd6-02af-4902-8d41-7728a44abfe5" />

<img width="495" height="549" alt="Image" src="https://github.com/user-attachments/assets/14f40c99-8b2f-4842-8d9a-2a4b19eec2f6" />

`mvn checkstyle:checkstyle` runs the **Checkstyle plugin** in Maven, which is used to check the source code for adherence to coding standards and best practices. This specific command tells Maven to execute the Checkstyle plugin and run the `checkstyle` goal, which analyzes the source code for any violations based on the defined Checkstyle rules. Update this script in `Jenkinsfile` to pipeline code and `Build Now`.

```groovy
stage('Checkstyle Analysis') {
    steps{
        sh 'mvn checkstyle:checkstyle'
    }
}
```

After running, Checkstyle generates a report on the violations found. The report is usually saved in the `target/checkstyle-result.xml` file (or another location if configured) which stores the output of Checkstyle’s code analysis in xml format which is not human readable. We have to send all the result to Sonar dashboard to do the analysis and translate the result.

!<img width="734" height="516" alt="Image" src="https://github.com/user-attachments/assets/5acdd9d0-6a7e-4705-bc74-120199d2a9e8" />

Since we're using the SonarScanner tool directly, we'll need to pass several parameters to upload various reports, such as Checkstyle, unit test results, and others, to the SonarQube server. These parameters ensure that all necessary reports are included in the analysis. From [SonarQube Scanner for Jenkins documentation](https://www.jenkins.io/doc/pipeline/steps/sonar/), we copy and edit the stage block. Update this pipeline code in `Jenkinsfile` and `Build Now`.

```groovy
stage('Sonar Code analysis') {
            environment {
                scannerHome = tool 'sonar6.2'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh '''${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=dummyApp \
                        -Dsonar.projectName=dummyApp \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/dummy/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

```
<img width="1290" height="157" alt="Image" src="https://github.com/user-attachments/assets/489b57c1-db73-41a4-aeb4-62448a563518" />

The project console shows that the results have been successfully uploaded to the SonarQube dashboard via the private IP. SonarQube creates a repository with the name `<sonar.projectName>` to store and display the analysis results.

<img width="1020" height="215" alt="Image" src="https://github.com/user-attachments/assets/00981dc9-20c3-4786-b51c-4949e9b0434c" />

## Quality Gate by SonarQube scanner

Create another rule in **Jenkins Security Group** that allows connection on port 8080 from SonarQube security group.

<img width="1759" height="537" alt="Image" src="https://github.com/user-attachments/assets/8652ff8b-73ab-459a-857d-2e1ea45a0638" />

In `Quality Gates` of SonarQube server, we get a default quality gate. We can create new quality gate instead of using the default quality gate in `Quality Gates` > `Create`. `Unlock editing` > `Add Condition` to add any metric(s).

<img width="668" height="454" alt="Image" src="https://github.com/user-attachments/assets/1d094cda-33b7-4d05-bb95-35d7a3b02232" />

To apply this quality gate to our project, go to `Project Settings` > `Quality Gate` then select our quality gate.

---

Once this quality gate is checked, SonarQube will contact Jenkins and send the result via webhook. We create webhook by selecting `Project Settings` > `Webhooks` > `Create`. Enter URL by `http://<Jenkins private IP>:8080/sonarqube-webhook`.

<img width="668" height="808" alt="Image" src="https://github.com/user-attachments/assets/e0f76a97-9c24-4707-8ea6-f0c8d8e62c7b" />

From [SonarQube Scanner for Jenkins documentation](https://www.jenkins.io/doc/pipeline/steps/sonar/), we copy and add another stage block. Update this pipeline code in `Jenkinsfile` and `Build Now`.

```groovy
stage("Quality Gate") {
    steps {
        timeout(time: 1, unit: 'HOURS') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

If the code doesn’t meet the quality gate condition, the job or the pipeline should fail due to quality gate failure.

<img width="1460" height="164" alt="Image" src="https://github.com/user-attachments/assets/50d070e2-c207-4152-ad3a-82182618d25c" />
<img width="1136" height="248" alt="Image" src="https://github.com/user-attachments/assets/094f1e19-3f52-4cf8-8c3c-3cbcb458097a" />


## Uploading Versioned Artifacts to NexusOSS Sonatype Repository

In `Manage Jenkins` > `Plugins` of Jenkins Dashboard, install `Nexus Artifact Uploader` and `Build Timestamp` plugins. Then, go to `http://<Nexus public IP>:8081` > `Server administration and configuration` > `Repositories` > `Create repository` , select `Maven2 (hosted)`. Name your repository and `create repository`.

<aside>
💡

In Nexus Repository Manager:

1. **Hosted Repository**: Stores internal artifacts you create (e.g., your company’s custom libraries or builds).
2. **Group Repository**: Aggregates multiple repositories (hosted and proxy) into one access point for easier management.
3. **Proxy Repository**: Caches external artifacts from remote repositories (e.g., Maven Central) to improve performance and reduce external dependencies.
</aside>

<img width="926" height="265" alt="Image" src="https://github.com/user-attachments/assets/ea8b57fd-634b-43a8-a187-bea83c664172" />

Set credentials Manage Jenkins > Credentials > System > Global credentials (unrestricted) > Create Credentials, then fill in Username and Password of Nexus account as well as name the credentials ID which will be use the create PAAC.

<img width="1718" height="874" alt="Image" src="https://github.com/user-attachments/assets/287fef9f-9aac-4bed-88a4-2b0ff20a0b1d" />

Update this pipeline code in `Jenkinsfile` and `Build Now`.

```groovy
stage("UploadArtifact"){
    steps {
        nexusArtifactUploader(
            nexusVersion: 'nexus3',
            protocol: 'http',
            nexusUrl: '172.31.91.146:8081', // private IP of Nexus server
            groupId: 'QA', // Ignore
            version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}", //dynamic env variable
            repository: 'dummy-repo', // repo that you created
            credentialsId: 'nexuslogin', // credential ID
            artifacts: [
                [artifactId:'dummyApp',  // name your artifact
                    classifier: '',  // Ignore
                    file: 'target/dummy-v2.war', // give path in workspace where the artifact is located
                    type: 'war']
            ]
        )
    }
}
```

Once Jenkins uploaded the artifact, the job is completed. In Nexus repository server, go to `Browse` > `<repository name>`. There are different versions of the artifact. Every time you run a pipeline you get another artifact.

<img width="1609" height="192" alt="Image" src="https://github.com/user-attachments/assets/a53e7c39-0136-4945-9ea6-1e4acc7f52d9" />

## Building Docker Image by Docker and Uploading to Amazon ECR

### Install Docker Engine and Change Permission

According to [Docker installation documentation](https://docs.docker.com/engine/install/ubuntu/), install `Docker Engine` in Jenkins SSH.

<img width="1290" height="465" alt="Image" src="https://github.com/user-attachments/assets/92e0f7d0-b3e6-4a4e-8b1c-26f893f0afd4" />

Since `jenkins` user execute everything in Jenkins job, running docker command as a Jenkins user would be denied. 

<img width="590" height="230" alt="Image" src="https://github.com/user-attachments/assets/e79d2ad8-9d5f-4b5b-b9f6-3eb415e8775e" />

We need to add the `jenkins` user to the `docker` group. Then, reboot the Jenkins to restart all services and reload all configuration.

<img width="1187" height="305" alt="Image" src="https://github.com/user-attachments/assets/8d1e4f33-3fd6-41a2-8c31-7ae8f59b5a85" />

### Create New Repository in Amazon ECR

Amazon (Elastic Container Registry) ECR is a container registry or a Docker Registry where we can store Docker images. To create a registry, go to `Amazon ECR` > `Private registry` > `Repositories` > `Create repository` and give registry a name.

<img width="1065" height="1004" alt="Image" src="https://github.com/user-attachments/assets/ec15faeb-ed87-49a9-b919-6f09ceb837c1" />

### Download Plugins in Jenkins

Download `Amazon Web Services SDK :: All`, `Amazon ECR`, `Docker Pipeline`, and `CloudBees Docker Build and Publish`.

<img width="1350" height="773" alt="Image" src="https://github.com/user-attachments/assets/b19ba6cb-70d8-4e57-8f8d-28120b00fdbe" />

### Create IAM User with Access Key

Go to `IAM` > `Access management` > `Users` > `Create user`. Give a name as `jenkins` and select  policies to allow this user accessing two services: `AmazonEC2ContainerRegistryFullAccess` to upload the Docker image to the registry and `AmazonECS_FullAccess` to deploy it on the Amazon ECS cluster.

<img width="1466" height="410" alt="Image" src="https://github.com/user-attachments/assets/4bda4ca7-8a4e-4cfd-ab8d-c0c172eb99f5" />

Click on the `jenkins` user > `Security credentials` > `Create access key` then select `Command Line Interface (CLI)` > `Create access key`. Download the CSV file containing access key and secret key.

<img width="1831" height="444" alt="Image" src="https://github.com/user-attachments/assets/14d5d8bb-3871-4367-8f0f-a0ce9d8e04b3" />

### Store Access Key and Secret Key in Jenkins Credentials

In `Manage Jenkins` > `Credentials` > `system` > `Global credentials (unrestricted)` > `Add Credentials`, select `AWS Credentials` which appears only after installing `Amazon Web Services SDK :: All` plugin, name your `<credentials_ID>` in ID, and fill in access key and secret key from your `CSV` file.

<img width="725" height="773" alt="Image" src="https://github.com/user-attachments/assets/53426e41-3d24-4824-8de1-f76342d6404e" />

### Update Pipeline Code

**Region:** Go to the browser where we created the registry, and copy the `<region>`.

<img width="328" height="52" alt="Image" src="https://github.com/user-attachments/assets/95913670-e299-4523-9919-0717c1f81297" />

**URI:** Go to `Amazon ECR` > `Private registry` > `Repositories`. Copy the `URI`.

<img width="1085" height="354" alt="Image" src="https://github.com/user-attachments/assets/27582be0-8ed2-4012-af61-87d20c63a72a" />

Update environment variables in the pipeline code.

```groovy
environment {
    // registryCredential = '<registry_type:region:credentials_ID>'
    registryCredential = 'ecr:us-east-1:awscreds' 
    // imageName = '<URI>'
    imageName = "<secret>.dkr.ecr.us-east-1.amazonaws.com/cicdproject"
    // registry = 'https://<URI without image_name>'
    registry = "https://<secret>.dkr.ecr.us-east-1.amazonaws.com" 
}
```

Since Jenkins does not have a built-in feature to build and upload Docker images to a registry, use the **`Docker Pipeline`** plugin. Update the pipeline code in the `Jenkinsfile` as following to include the necessary Docker build and push steps, and then trigger the pipeline using **Build Now**.

1. **Docker Image Build**: The pipeline first triggers the Docker image build using the `docker.build` command, where it uses a multistage Dockerfile to create an image. The Docker image is built and stored in the `dockerImage` variable. The image is tagged with the build number `$BUILD_NUMBER` and `latest` tag.

```groovy
stage('Build App Image') {
steps {
    script {
        // Run docker plugin with build function: docker.build("URI"+":<image_tag>", "<Dockerfile path from the Github source code>")
        dockerImage = docker.build(imageName + ":$BUILD_NUMBER", ".")
    }

}
```

2. **Push Docker Image**: After the image is built, Jenkins logs into the Amazon Elastic Container Registry (ECR). This step authenticates Jenkins to push the image to the specified ECR repository, ensuring secure access with AWS credentials. The image layers are pushed to the Amazon ECR registry using the `docker push` command. Both tags are pushed to the repository, making the image available for deployment.

```groovy
stage('Upload App Image') {
    steps{
        script {
            // Run docker plugin with withRegistry function: withRegistry("https://<URI without image_name>'", "<registry_name:region:credentials_ID>") for Docker plugin to login to registry
            docker.withRegistry(registry, registryCredential) {
                // Push Docker image with the tags
                dockerImage.push("$BUILD_NUMBER") // tag1
                dockerImage.push('latest') // tag2
            }
        }
    }
}
```

3. **Cleanup Docker Images:** To prevent disk space from being consumed by remaining Docker image in workspace of Jenkins server over time, add a stage to remove the image after it has been successfully uploaded. This ensures that disk space is freed up with each Docker image build.

```groovy
stage('Remove Container Images'){
    steps{
        sh 'docker rmi -f $(docker images -a -q)'
    }
}
```

After a successful job, in `Amazon ECR` > `Private registry` > `Repositories`, clicking on the repository will navigate to the uploaded image storing the artifacts in the ECR repository.

<img width="1498" height="222" alt="Image" src="https://github.com/user-attachments/assets/dfef284f-18a8-4a64-9cf5-641531a60dc6" />

---

# Continuous Deployment Flow

## Deployment by Amazon ECS

### Create CLusters

An **ECS Cluster** in Amazon Elastic Container Service (ECS) is a logical grouping of resources where containers are deployed, managed, and run. It acts as the foundation for hosting and managing tasks (containers) and services. Go to `Amazon Elastic Container Service` > **`Clusters` > `Create cluster` to create a cluster.**

<aside>
🚨

AWS ECS offers multiple deployment options:

1. **AWS Fargate**: A serverless option where AWS manages compute resources, scaling, and maintenance, providing zero overhead for cluster management. Fargate is the simplest and most hassle-free approach.
2. **EC2 Instances**: Requires setting up an auto-scaling group, specifying instance limits, and managing capacity, offering more control but higher overhead.
3. **ECS Anywhere**: Allows integration of external resources into the ECS cluster.
</aside>

<img width="1491" height="1086" alt="Image" src="https://github.com/user-attachments/assets/9f10c659-9586-4a10-aa3b-ae6ef4b300d9" />

### Create Task Definition

A **task definition** in ECS specifies all the essential details for running a container, including where is registry from, the Docker image URI where to pull the Docker image, CPU, RAM, port mappings, and IAM roles. It acts like an EC2 AMI but is tailored for containers. To create one, navigate to **`Amazon ECS` > `Task definitions` > `Create new task definition`**, provide a name, and input the Image URI from the ECR registry. This task definition is then used to launch and manage containers within your ECS cluster.

<aside>
🚨

If the specific Docker image runs on Tomcat, the port mapping container is 8080.

</aside>

<img width="1324" height="1196" alt="Image" src="https://github.com/user-attachments/assets/7c6be183-7be3-4c38-ac29-5c5e643b4923" />

In `Create new task definition`, `task roles` is the IAM role that this ECS service uses to access other AWS services. `Task execution role` will create new role for us. 

<img width="934" height="191" alt="Image" src="https://github.com/user-attachments/assets/f88a6e75-d099-4784-afc9-f30c37499193" />

By default, the IAM role associated with a task definition lacks permissions to access CloudWatch.

<img width="1085" height="419" alt="Image" src="https://github.com/user-attachments/assets/2d7d9e1a-4f3a-4eca-82cd-29d4df188717" />

To enable log storage in CloudWatch, after creating task definition, navigate to **`Task definition` > `<Task definition name>` > `ecsTaskExecutionRole` > `Add permission`**, select **`Attach policies`**, and attach the **`CloudWatchLogsFullAccess`** policy to the role. This step is crucial for storing container logs in CloudWatch.

<img width="837" height="287" alt="Image" src="https://github.com/user-attachments/assets/4acd8a59-31b6-4294-8e53-4aef696182fe" />
<img width="1478" height="275" alt="Image" src="https://github.com/user-attachments/assets/747eb044-f1be-43bd-8a76-2905fdab9b02" />
<img width="1849" height="642" alt="Image" src="https://github.com/user-attachments/assets/3b96b57b-0783-4db3-9887-1fc391433a44" />

### Create Service

In ECS, a **task** represents a container running based on a task definition. While you can directly run a task (container), managing it manually—such as starting, stopping, or restarting on failure. Instead, ECS offers **`services`**, which automate container management. `Services` ensure tasks run continuously, handle scaling, and can integrate with load balancers for efficient traffic distribution. It is the job of service to manage containers. To create a service, navigate to `Clusters` > `<clusterName>` > `Services` > `Create`.

- **Application type**: Select **Service** for continuously running applications like Tomcat or MySQL. Choose "Task" for short-lived or one-time processes.
- Family: Choose the previously created task definition containing container details.
- Revision: Choose version of the task definition.
- **Desired Tasks**: Specify the number of tasks (containers) to run. Start with one to minimize costs but can scale later for high availability.
- **Deployment option**: Select how many you want container to be upgraded at a time when you have multiple containers to run.
- **Deployment failure detection:** Uncheck deployment failure detection initially to avoid rollback complexities.

<img width="1474" height="1163" alt="Image" src="https://github.com/user-attachments/assets/c6444a0b-df72-4ec9-854d-20b5c59e02df" />

- **Networking**:
- Use the **default VPC** and **subnets**.
- Create a new security group **for the ELB and the container** with:
- **HTTP (port 80)**: Open for your public access to the load balancer URL.
- **Custom TCP (port 8080)**: Open for the load balancer to route the request to the container on port 8080 where container runs the service.


<img width="1460" height="958" alt="Image" src="https://github.com/user-attachments/assets/81db356c-c9e5-4ba5-9168-4f5a03ff8467" />

- **Load Balancer**:
- Use an **Application Load Balancer (ALB)**.
- Container: `<Backend port of the ELB>: <Frontend port of the container>` which is `8080:8080`.
- Set up the load balancer to:
- listens on **port 80** (frontend).
- routes requests on **port 8080** (backend).


<img width="1237" height="937" alt="Image" src="https://github.com/user-attachments/assets/0857cadb-9585-4a41-a725-43c795fb19d3" />

After service is created, click on the `<serviceName>` > `Health and metrics` > `Configuration and networking` > `Configuration and networking` >`DNS names` where you can access through the browser.

<img width="2170" height="367" alt="Image" src="https://github.com/user-attachments/assets/44e2cd0a-ead3-493b-bcba-166b244d244d" />

`Amazon Elastic Container Service` > `Clusters` > `<clusterName>` > `Tasks` > `<taskName>` > `Logs` is logs generated by the container.

<img width="1955" height="482" alt="Image" src="https://github.com/user-attachments/assets/76102127-8d6e-471e-bfa9-e21e7304750e" />

### Install AWS CLI

Installing AWS CLI in Jenkins SSH to run AWS CLI commands on Jenkins.

<img width="975" height="78" alt="Image" src="https://github.com/user-attachments/assets/35381165-6aca-42a2-8c84-b82378a4f646" />

<aside>
🚨

Make sure you give `jenkins` user a policy `AmazonECS_FullAccess` to deploy it on the Amazon ECS cluster in `IAM` > `Access management` > `Users`.

</aside>

### Download Plugins

In `Manage Jenkins` > `Plugins`, download `Pipeline: AWS Steps`.

### Update Jenkinsfile

We fill in the cluster and service information into the pipeline script as global variable.

```groovy
environment {
    // Create ECS cluster containing service. Service is a task which will run your container fetching image from ECR.
    // cluster = '<cluster_name>'
    cluster = 'cicdprojectcluster'
    // service = '<service_name>'
    service = 'cicdprojectsvc'
}
```

1. **AWS Credentials and Region Setup**:
    - `withAWS(credentials:'awscreds', region: 'us-east-1')`: Configures the AWS CLI to use specific credentials and region for the commands.
2. **Update ECS Service**:
    - The command `aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment`:
    - Forces the ECS service to deploy a new container by replacing the old one. The old container will be deleted.
   - Fetches the latest container image (based on the ECS task definition).

Update this pipeline code in `Jenkinsfile` and `Build Now`.

```groovy
stage('Deploy to ECS'){
    steps{
        // withAWS(credentials: '<aws_credentials_ID>', region: '<region>')
        withAWS(credentials:'awscreds', region: 'us-east-1'){
            // AWS CLI to remove the old container with the old image tag, fetch new image with the latest tags, and run it on the ECS cluster.
            sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
        }
    }
}
```

A new task was created while the old task remained active until the new one became fully operational. Once the new task became **100% healthy**, the old task was decommissioned.

<img width="769" height="296" alt="Image" src="https://github.com/user-attachments/assets/222b3c26-0414-4350-b15c-39f927156939" />
<img width="639" height="352" alt="Image" src="https://github.com/user-attachments/assets/b41b9e26-a1d4-4300-a7f0-60f6df83876e" />

Finally, you have only one task after the previous task got stopped. 
The container ID will confirm that it is a new container.

<img width="768" height="248" alt="Image" src="https://github.com/user-attachments/assets/3513a2e8-a3e6-46db-94a7-75f55d81b26b" />
<img width="867" height="353" alt="Image" src="https://github.com/user-attachments/assets/bef89bb3-b784-4fca-8ad9-d2da69f992e6" />

## Sending Notification by Slack

In `Manage Jenkins` > `Plugins`, install `Slack Notification` or any Notification tool of your choice. Then, login to Slack webpage. Create workspace and channel.

<img width="989" height="403" alt="image" src="https://github.com/user-attachments/assets/e4cbd131-2cf6-493b-8536-31936354b843" />

To integrate Jenkins with Slack account, we need a token that will get to Jenkins. In `Add Apps to Slack` , search for `Jenkins CI` and `Add to Slack`. Choose your channel to `Add Jenkins CI integration`.

<img width="1508" height="643" alt="Image" src="https://github.com/user-attachments/assets/78061d6c-4e94-4e0d-b001-040e17edb121" />

Copy the token in Integration Settings, and in Slack app, copy `<workspace_name>` from `https://<workspace_name>.slack.com`. In Jenkins server, go to `Manage Jenkins` > `System` > `Slack`, and paste workspace name in `Workspace`, token in `Add Credentials`, and channel name as `#<channel_name>` in `Default channel / member id`.

<img width="1759" height="933" alt="Image" src="https://github.com/user-attachments/assets/51ee82ab-048f-4f68-9845-e70094fc9d57" />

Add another function to indicate the colors corresponding to its status.

```groovy
def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]
```
The notification is not any extra stage but post-installation of your pipeline setting notification of entire pipeline. Therefore, it is going to be outside of `stages` block.

```groovy
post {
		always {
				echo 'Slack Notifications.'
				slackSend channel: '#devopscicd',
				color: COLOR_MAP[currentBuild.currentResult],
				message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
		}
}
```

Update this pipeline code in `Jenkinsfile` and `Build Now`.

<img width="559" height="276" alt="Image" src="https://github.com/user-attachments/assets/0537a1d8-c4f6-4b9c-a5e0-4d734230bd44" />
