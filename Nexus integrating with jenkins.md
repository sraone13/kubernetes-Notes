Artifact : It is used to store the log(var/jar) files for our project.


Nexus
S3
JFrog

Sonar cube: it si used for the code test( Code Quality).
It is used by test engineers



Nexus:


1. Create and Instance( Jenkins configuration)
2. In server setup the Jenkins (using bash, all jenkins setup commands in paste in one bash file and execute)
	sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
	sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
	amazon-linux-extras install java-openjdk11 -y
	yum install jenkins -y && systemctl restart jenkins
	login to the jenkins Dashboard.
	
3.	yum install java-1.8.0-openjdk -y
	yum install maven -y
	yum install git -y
	
Vim /etc/maven/settings.xml (line-119)

<server>
  <id>nexusRepo</id>
  <username>admin</username>
  <password>123456</password>
</server>

update-alternatives --config java  (select the 11 version java becasue to install the plugins)

4. Install "nexus platform plugin" in jenkins from the avaiable plugins.

5.In Jenkins select new job(free style) -->github project(give the gitbub url) --->Source code management(git)---> 
	branch (master) ---> Build steps (Invoke top level maven targets) ---> Goals (clean package or clean install) ---> Save it and bulid now.

6.Now cross check the war file is created or not by cd /var/lib/jenkins/workspace/one/target  ---> ll

7.Now we need to integrate the nexus server with Jenkins server.
Go to jenkins —> configuration system —>sonatype nexus -->select(Nexus repository manager 3X server)---> 
Display name: nexus
Server-id: nexus-repo
Server url: public ip of nexus:8081
Credentials: username and password of the nexus
Test connection and save.

8.Go to job-->Configure —>

Add a build step: Nexus Repository Manager Publisher
Nexus instance: auto update
Nexus-repo: sample-releases

packages: 
Group: search in pom.xlm
Artifact id: search in pom.xlm
Version: search in pom.xlm
Package: war

Maven Artifact:
File path: target/myweb-8.2.5.war (check the version in pom.xml and war file and update here)
Extension: .war


9.Now click on bulid now and see the war file will in Nexus server.

10. When devepler commits the code we need to change the version and in file path also change and build now.
 
 
Or or or or or 

Install "nexus artifact uploader" in jenkins from the avaiable plugins.

Go to job —>configure
Add build step —> nexus artefact uploader
Nexus version: nexus3
Protocol: HTTP
Nexus URL: ipaddress:8081
crenentials: ****
Group id: see in pom.xml
Version: see in pom.xml
Repository: sample-releases


artifact:
Artifact id: search in pom.xlm
Type: war
fle: target/myweb.war



Nexus Setup:

1.Create one Instance with 1 CPU and 2GB of RAM(t2.small)
	In Security group along with ssh
	add one
	Type:all traffic
	Source type: Any where
	
	add one more:
	Type: Custom TCP
	Source type: Any where
	Port range: 8081

2. sudo yum update -y
   sudo yum install wget -y
   sudo yum install java-1.8.0-openjdk.x86_64 -y
   sudo mkdir /app && cd /app
   sudo wget -O nexus.tar.gz https://download.sonatype.com/nexus/3/latest-unix.tar.gz
   sudo tar -xvf nexus.tar.gz
   ll
   sudo mv nexus-3* nexus
   ll
   sudo mv nexus-3* nexus
   sudo adduser nexus
   sudo chown -R nexus:nexus /app/nexus or chown nexus:nexus nexus -R
   sudo chown -R nexus:nexus /app/sonatype-work or chown nexus:nexus sonatype-work -R
   sudo vi  /app/nexus/bin/nexus.rc 

run_as_user="nexus"  ---> save it :wq

   sudo vi /app/nexus/bin/nexus.vmoptions
   cd ../../../ (go root user)
   
   
3.sudo vi /etc/systemd/system/nexus.service  (In this file add below content as it is and save it)		

[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
User=nexus
Group=nexus
ExecStart=/app/nexus/bin/nexus start
ExecStop=/app/nexus/bin/nexus stop
User=nexus
Restart=on-abort

[Install]
WantedBy=multi-user.target

4.  chkconfig nexus on
	systemctl start nexus
   systemctl status nexus.service 
   systemctl enable nexus
   systemctl status nexus.service 
   history
   
5. Now copy the public ip address and paste in browser with:8081 port number.
6. Click on sign and give user as admin and password get from the give path and paste here.
7. add new password

8. select the setting and select the repositories ---> Create a repository -->select recipe(maven hosted 2) ---> repository should be with pom.xml
   (get the pom.xml from the gitbub.com/devops0014/one/blob/master/pom.xml to sravan repository for you future use)
   
   whatever the name should have repository in pom.xml with same name you have to create the repository in Nexus.--> Create repository
   
9.Go to browse and click on the sample-release repository(no var file can see now)

10. In github in pom.xml update the public ip address with nexus public ip address and commit and save.

	


------------------------------------------------------------------------------------------------------------------------------------------


JFrog with Jenkins:

1. Create an Instance with t2.medium(AMI: LINUX AMI INSTANCE: T2 MEDIUM S.G : 8081 & ANYWHERE)  and securty groug(take the nexus group)
	Configure storage: 15 Gb of SSD

2.INSTALLATION:

wget https://releases.jfrog.io/artifactory/artifactory-rpms/artifactory-rpms.repo -O jfrog-artifactory-rpms.repo
mv jfrog-artifactory-rpms.repo /etc/yum.repos.d/
yum update -y
yum install jfrog-artifactory-oss -y
systemctl start artifactory.service 
systemctl enable artifactory.service 
systemctl status artifactory.service


3.Now copy the public ip address and paste in browser with:8081 port number.
creds: admin
password: password

Create new password --->next--->skip-->skip--->Creat repository(Maven)--->Next--->Finish.

4.Create repository--->Local repository--->java app(maven) --->repository key (java-app) --->Create repository.

5.You can the artifactory and repositories you created.

6. Go to Jenkins and install the plugin (Artifactory) 

7.Go to configure system;

JFrog
Add JFrog platform instance
Instance id : Jfrog
JFrog url: ipaddress:8081
Username:admin
password: give the password of JFrog

Test the Connection.

8. Now i need to integrate with git and jenkins like the Nexus did......
