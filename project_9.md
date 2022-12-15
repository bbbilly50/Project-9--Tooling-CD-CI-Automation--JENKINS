## **TOOLING WEBSITE DEPLOYMENT AUTOMATION WITH CONTINUOUS INTEGRATION. INTRODUCTION TO JENKINS**
---
---

</br>

### **INSTALL AND CONFIGURE JENKINS SERVER**

</br>

**Step 1 – Install Jenkins server**

</br>

1. Create an AWS EC2 server based on Ubuntu Server 20.04 LTS and name it `"Jenkins"`

2. Install `JDK` (since Jenkins is a Java-based application)

`sudo apt update`

`sudo apt install default-jdk-headless`

3. Install Jenkins

</br>

```py
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt-get install jenkins
```
Make sure Jenkins is up and running

`sudo systemctl status jenkins`

![jenkins-status](./images-project9/jenkins-status.PNG)

</br>

4. By default `Jenkins server uses TCP port 8080` – open it by creating a new Inbound Rule in the EC2 Security Group
   
![open-8080](./images-project9/open-8080.PNG)
   
</br>

1. Perform initial Jenkins setup.
From your browser access 

`http://<Jenkins-Server-Public-IP-Address-or-Public-DNS-Name>:8080`

</br>

![getting-started](./images-project9/geting-started.PNG)

You will be prompted to provide a default admin password

Retrieve it from your server:

`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`

Then you will be asked which plugings to install – choose `suggested plugins`.

Once plugins installation is done – create an admin user and we will get the Jenkins server address.

![jenkins-setup](./images-project9/jenkins-setup.PNG)

</br>

![jenkins](./images-project9/jenkins.PNG)

---
---

</br>

### **Step 2 – Configure `Jenkins` to retrieve `source codes` from `GitHub using Webhooks`**

</br>

Configure a  `Jenkins job/project`  This job will will be `triggered` by `GitHub webhooks` and will `execute a ‘build’ task` to retrieve codes from GitHub and store it locally on Jenkins server.

1. Enable webhooks in your GitHub repository settings

</br>

![adding-webhook](./images-project9/adding-webhooks.PNG)

</br>

![webhooks](./images-project9/webhooks.PNG)

</br>

2. Go to Jenkins web console, click `"New Item"` and create a `"Freestyle project"`
   
   ![new-item](./images-project9/new-item.PNG)

To connect your GitHub repository, we will need to provide its URL, we can copy from the repository itself

In configuration of `Jenkins freestyle project` choose `Git repository`, provide there the link to the Tooling GitHub repository and credentials (user/password) so Jenkins could access files in the repository.

Note: Set `Branch Specifier (blank for 'any')?`  to 

`**/main` 

to mach the `master branche` name in github.


`Save the configuration` and let us try to run the build. For now we can only do it manually.

</br>

![git-url](./images-project9/branch%20specifier.PNG)

Click `"Build Now"` button, if you have configured everything correctly, the build will be successfull and you will see it under #1

![build](./images-project9/build.PNG)

This build does not produce anything and it runs only when we trigger it manually. Let us fix it.

3. Click `"Configure" your job/project` and add these two configurations

   - **Configure triggering the job from GitHub webhook:**

</br>

![git-triger](./images-project9/git-triger.PNG)

</br>

   - **Configure `"Post-build Actions"` to archive all the files – files resulted from a build are called `"artifacts"`.**

![post-build](./images-project9/post-build.PNG)

</br>
Now, let make some change in any file in our GitHub repository (e.g. README.MD file) and push the changes to the master branch.

We will see that a new build has been launched automatically (by webhook) and we can see its `results – artifacts, saved on Jenkins server`.

</br>

![webhook-jenkins](./images-project9/jenkins-webhook.PNG)

By default, the artifacts are stored on Jenkins server locally

`ls /var/lib/jenkins/jobs/tooling_github/builds/<build_number>/archive/`

---
---
</br>

### **Step-3 CONFIGURE JENKINS TO COPY FILES TO NFS SERVER VIA SSH**

</br>

1. **Install "Publish Over SSH" plugin.**
   
   On main dashboard select "Manage Jenkins" and choose "Manage Plugins" menu item.

   On `"Available"` tab search for `"Publish Over SSH" plugin` and install it

</br>

   ![push-ssh](./images-project9/push-ssh.PNG)

2. **Configure the job/project to copy artifacts over to NFS server.**
   
   On main dashboard select `"Manage Jenkins" `and choose `"Configure System"` menu item.

   Scroll down to `Publish over SSH plugin` configuration section and configure it to be able to connect to your NFS server:

   1. Provide a `private key` (content of .pem file that you use to connect to NFS server via SSH/Putty)
   
   2. `Arbitrary name`

   3. `Hostname `– can be private IP address of your NFS server
   
   4. `Username – ec2-user `(since NFS server is based on EC2 with RHEL 8)

   5. `Remote directory – /mnt/apps` since our Web Servers use it as a mounting point to retrieve files from the NFS server
 
![publish-over-ssh](./images-project9/publish-over-ssh.PNG)

![publish-over-ssh](./images-project9/publish-over-ssh2.PNG)

3. `Test the configuration` and make sure the connection returns Success. Remember, that `TCP port 22 `on `NFS server` must be open to receive SSH connections.

![configuration-test](./images-project9/configuration-test.PNG)

`Save the configuration,` open your Jenkins job/project configuration page and `add` another one `"Post-build Action"`:

`"Send build artifacts over SSH"`

![send-build-artifacts](./images-project9/send-build-artifacts.PNG)

</br>

Configure it to `send all files probuced by the build` into our previouslys define remote directory. In our case we want to copy all files and directories – so we use : `**`

![send-files](./images-project9/send-files.PNG)

If you want to apply some particular pattern to define which files to send – use this syntax.

Save this configuration and go ahead, change something in `README.MD file in your GitHub` Tooling repository.

`Webhook will trigger` a new job and in the `"Console Output"` of the job you will find something like this:

      SSH: Transferred 25 file(s)
      Finished: SUCCESS

</br>

![NFS-connection](./images-project9/NFS%20connection.PNG)
To make sure that the files in `/mnt/apps` have been udated – connect via `SSH/Putty` to your NFS server and check `README.MD` file

`cat /mnt/apps/README.md`

![readme-updated](./images-project9/readme-updated.PNG)