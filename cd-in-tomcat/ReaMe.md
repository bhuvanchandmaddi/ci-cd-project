## Step by step instructions to create CI/CD on vm


### Steps on machine 1

![Flow](https://github.com/bhuvanchandmaddi/ci-cd-project/blob/main/.images/tomatserverflow.PNG?raw=true)

1. Install the Java and then install the tomcat(Used 9 in testing)
    ```
    $amazon-linux-extras -> Lists all the packages i.e java,epel etc

    $amazon-linux-extras install java-openjdk11 -> Installs openjdk11
    ```

1. Launch 2 EC2 instances(one for Jenkins server and other for installing tomcat)

1. Connect to first ec2 instance and install the jenkins Server by following [these](https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/) instructions

1. Configuring jenkins don't install any plugins literally zero(even git,maven etc)


1. Install Git in ec2 instance, then install **Git plugin** and configure the same in Jenkins by visiting **Global tool configuration page**

1. Install the maven(using [these](https://github.com/yankils/Simple-DevOps-Project/blob/master/Jenkins/maven_install.MD) instructions)

1. Install **Maven Integration plugin** and configure the same in Jenkins by visiting **Global tool configuration page**

1. Configure the Java location in the same page i.e **Global tool configuration page**


### Steps on machine 2

1. Install the Java and then install the tomcat(Used 9 in testing)
    ```
    $amazon-linux-extras -> Lists all the packages i.e java,epel etc

    $amazon-linux-extras install java-openjdk11 -> Installs openjdk11
    ```
1. Then install tomcat and then do [configuration changes](https://github.com/yankils/Simple-DevOps-Project/blob/master/Tomcat/tomcat_installation.MD)

### Create a CI pipeline

* Fork this [project](https://github.com/yankils/hello-world)

* Create a maven project and specify the above github details under git section

* Under maven goals, specify **clean install**, which creates new artifact

* Test it by click on build

### Add CD in above pipeline

* Install **Deploy to container** plugin

* Update the Post build actions -> Deploy War/Ear to a container.

```
WAR/EAR files:
**/*.war
```
* For containers sections, enter jenkins url and deployer role creds(deloyer and deployer in our case)

* Finally test the build.

* The war should be deployed into tomcat

### Additional tasks

* [Create a web-hook](https://www.blazemeter.com/blog/how-to-integrate-your-github-repository-to-your-jenkins-project), so that jenkins pipeline will be trigged when a check in was made to github project.




