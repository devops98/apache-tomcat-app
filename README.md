Role Name
=========

hear i created three roles apache (in apache role i am going to install apache2,libapche2-mod-jk and openjdk-7-jdk) ,tomcat and mysql roles.

Requirements
------------

we nee ansible server and remote server. control server connect with directly remote server root user when connect with openssh.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }


execution Information
------------------
Enter configuraed remote server host or ip in env inventory file


      ansible-playbook -i env main.yaml


Manual Steps
----------------

install Apache2
***************

We are going to use Ubuntu's APT package maintenance system to obtain and install Apache2.

sudo apt-get install apache2

install and install mod_jk
**************************

The mod_jk module is not included in the Apache2 download so must be obtained and installed separately. 
The installation requires that the mod_jk module is visible to Apache and configured to ensure that Apache 
knows where to look for it and what to do with the requests you want to proxy.

sudo apt-get install libapache2-mod-jk

This will install in /etc/libapache2-mod-jk also two files have been added to the /etc/apache2/mods-available folder.

installing Tomcat 7
*******************

At the time of writing this Tomcat 7 does not have a package in APT so you must download the binaries from the tomcat
website.http://www-us.apache.org/dist/tomcat/tomcat-7/v7.0.79/bin/apache-tomcat-7.0.79.tar.gz
select the appropriate binary distribution and extract it as follows.

tar xvzf apache-tomcat-8.0.5.tar.gz

install Java
*************
apt-get install openjdk-7-jdk

Path setting
************
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64


Configure two local Tomcat servers
**********************************
We will edit only the server.xml of the server2 installation of tomcat. We need to change port numbers to avoid conflicts.We change the following:



<Server port="9005" shutdown="SHUTDOWN">
<Connector port="9009" protocol="AJP/1.3" redirectPort="9443"/>


and comment out the HTTP Connector as we only want the web application to be accessible through the load balancer.
Here is my server2 Tomcat server.xml configuration.

Configuring Apache Tomcat for MySQL manually
********************************************
Procedure
*********
Download the binary distribution appropriate for your platform, extract the JAR file, and copy it to both tomcat servers "$CATALINA_HOME/lib". 
This makes the driver available to JDBC.

https://dev.mysql.com/downloads/connector/j/
https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.43.tar.gz

edit server.xml file
********************

Prepare an XML statement that defines the data source, as shown in the following code example. 
Insert this statement in the server.xml file, as indicated in Configuring Apache Tomcat manually.

    <Resource
        name="jdbc/javatestdb" type="javax.sql.DataSource"
        maxActive="100" maxIdle="30" maxWait="10000" 
        url="jdbc:mysql://localhost:3306/javatestdb"
        driverClassName="com.mysql.jdbc.Driver"
        username="root" password="root"
    />




Configure mod_jk
****************
Load balancing is configured in the workers.properties file, located /etc/libapache2-mod-jk/ where workers represent actual or virtual workers.
We will define two actual workers and two virtual workers which map to the Tomcat servers. In the worker.list property 
I have defined two virtual workers: status and loadbalancer, I will refer to these later in the Apache configuration.
Workers for each server have been defined using values for the server.xml configuration files. I used the port values for the AJP connectors and 
I have included an lbfactor that sets the preference that the load balancer will show for that server.Finally we define the virtual workers. 
The loadbalancer worker is set to type lb and set the workers that represent the Tomcat servers in the balancer_workers properties.
The status only needs to be set to type status.

worker.list=loadbalancer,status  
worker.server1.port=8009
worker.server1.host=localhost
worker.server1.type=ajp13  
worker.server2.port=9009
worker.server2.host=localhost
worker.server2.type=ajp13  
worker.server1.lbfactor=1
worker.server2.lbfactor=1  
worker.loadbalancer.type=lb
worker.loadbalancer.balance_workers=server1,server2  
worker.status.type=status

Ensure that you remove any other worker configuration that are not being used.


Configure Apache Web Server to forward requests
************************************************

You will need to add the following to the Apache configurations located in etc/apache2/sites-enabled/000-default.conf

JkMount /status status
JkMount /* loadbalancer

Verify the installation
***********************

To test that all has been configured correctly we need to deploy an application.
A sample application that has been used for years to test such configurations is called the ClusterJSP sample application. 
You can find it by googling in or from the JBoss site.Now deploy the war to the webapps folder on both servers and start each server using 
the start-up script /opt/tomcat-server1/bin/startup.sh.Go to http://localhost/clusterjsp/HaJsp.jsp and you should see the page show HttpSession information.

Now lets look at the mod_jk status page: http://localhost/status. 
You will see that this page shows information about the load balancer workers and the workers it is balancing.
