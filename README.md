# Jenkins Automated deployment


#### Inspired by https://technologyconversations.com/2017/06/16/automating-jenkins-docker-setup/
#### Plugins inspired by :
https://github.com/fabric8io/jenkins-docker/blob/master/plugins.txt
https://caylent.com/jenkins-plugins/

#### Uses Alpine image as base

#### Creating Jenkins Image With Automated Setup
Before we define Dockerfile that we’ll use to create Jenkins image with automatic setup, we need to figure out how to create at least one administrative user. Unfortunately, there is no (decent) API we can invoke. Our best bet is to execute a Groovy script that will do the job. The script is as follows.

```
#!groovy
 
import jenkins.model.*
import hudson.security.*
import jenkins.security.s2m.AdminWhitelistRule
 
def instance = Jenkins.getInstance()
 
def hudsonRealm = new HudsonPrivateSecurityRealm(false)
hudsonRealm.createAccount("admin", "admin")
instance.setSecurityRealm(hudsonRealm)
 
def strategy = new FullControlOnceLoggedInAuthorizationStrategy()
instance.setAuthorizationStrategy(strategy)
instance.save()
 
Jenkins.instance.getInjector().getInstance(AdminWhitelistRule.class).setMasterKillSwitch(false)
```

The key is in the hudsonRealm.createAccount("admin", "admin") line. It creates an account with both username and password set to admin.

The major problem with that script is that it has the credentials hard-coded. We could convert them to environment variables, but that would be very insecure. Instead, we’ll leverage Docker secrets. When attached to a container, secrets are stored in /run/secrets in-memory directory.

More secure (and flexible) version of the script is as follows.

```
#!groovy
 
import jenkins.model.*
import hudson.security.*
import jenkins.security.s2m.AdminWhitelistRule
 
def instance = Jenkins.getInstance()
 
def user = new File("/run/secrets/jenkins-user").text.trim()
def pass = new File("/run/secrets/jenkins-pass").text.trim()
 
def hudsonRealm = new HudsonPrivateSecurityRealm(false)
hudsonRealm.createAccount(user, pass)
instance.setSecurityRealm(hudsonRealm)
 
def strategy = new FullControlOnceLoggedInAuthorizationStrategy()
instance.setAuthorizationStrategy(strategy)
instance.save()
 
Jenkins.instance.getInjector().getInstance(AdminWhitelistRule.class).setMasterKillSwitch(false)
```

This time, we placed contents of jenkins-user and jenkins-pass files into variables and used them to create an account.

Please save the script as security.groovy file.

Equipped with the list of plugins stored in plugins.txt and the script that will create a user from Docker secrets, we can proceed and create a Dockerfile.

Please create Dockerfile with the content that follows.

```
FROM jenkins/jenkins:lts-alpine
 
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"
 
COPY security.groovy /usr/share/jenkins/ref/init.groovy.d/security.groovy
 
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt
```

The image will be based on alpine version of Jenkins which is much smaller and more secure than others.

We used JAVA_OPTS to disable the setup wizard. We won’t need it since our setup will be fully automated.

Next, we’re copying the security.groovy file into /usr/share/jenkins/ref/init.groovy.d/ directory. Jenkins, when initialized, will execute all the Groovy scripts located in that directory. If you create additional setup scripts, that is the place you should place them.

Finally, we are copying the plugins.txt file that contains the list of all the plugins we need and passing that list to the install-plugins.sh script which is part of the standard distribution.

With the Dockerfile defined, we can build our image and push it to Docker registry.

Please replace vfarcic with your Docker Hub user.

```
docker image build -t vfarcic/jenkins .
 
docker login # Answer the questions with your credentials
 
docker image push vfarcic/jenkins
```

With the image built and pushed to the Registry, we can, finally, create a Jenkins service.

Creating Jenkins Service With Automated Setup
We should create a Compose file that will hold the definition of the service. It is as follows.

Please replace vfarcic with your Docker Hub user.

```
version: '3.1'
 
services:
 
  main:
    image: vfarcic/jenkins
    ports:
      - 8080:8080
      - 50000:50000
    secrets:
      - jenkins-user
      - jenkins-pass
 
secrets:
  jenkins-user:
    external: true
  jenkins-pass:
    external: true
```

Save the definition as jenkins.yml file.

The Compose file is very straightforward. If defines only one service (main) and uses the newly built image. It is exposing ports 8080 for the UI and 50000 for the agents. Finally, it defines jenkins-user and jenkins-pass secrets.

We should create the secrets before deploying the stack. The commands that follow will use admin as both the username and the password. Feel free to change it to something less obvious.
```
echo "admin" | docker secret create jenkins-user -
 
echo "admin" | docker secret create jenkins-pass -
```
Now we are ready to deploy the stack.

```
docker stack deploy -c jenkins.yml jenkins
```
A few moments later, the service will be up and running. Please confirm the status by executing docker stack ps jenkins.

Let’s confirm that everything works as expected.

1
open "http://localhost:8080"
You’ll notice that this time, we did not have to go through the Setup Wizard steps. Everything is automated and the service is ready for use. Feel free to log in and confirm that the user you specified is indeed created.

If you create this service in a demo environment (e.g. your laptop) now is the time to remove it and free your resources for something else. The image you just built should be ready for production.

```
docker stack rm jenkins
 
docker secret rm jenkins-user
 
docker secret rm jenkins-pass
```

### To enable multiple executors, need to change in Manage Jenkins==> COnfigure Systems==># of executors =10
