## Beginning  
#1. Update the package list and install Nginx:
```
sudo apt update  
sudo apt install nginx -y
```
#2. Start Nginx and enable it to run on boot:
```
sudo systemctl start nginx
sudo systemctl enable nginx
```
#3. Set Up Website Directory: Create a directory for your website files:
```
sudo mkdir -p /var/www/mywebsite
sudo chown $USER:$USER /var/www/mywebsite
```
#4. Configure Nginx to serve this directory by editing the default site configuration:
```
sudo vi /etc/nginx/sites-available/default
```
#5. Update the root directive to point to your directory:
```
root /var/www/mywebsite;
```
#6. Save and exit, then restart Nginx:
```
sudo systemctl restart nginx
```
#7. Test the Web Server: Place a sample index.html file in /var/www/mywebsite:  
```
cd /var/www/mywebsite  
vi index.html [<h1>Hello from Azure VM!</h1>]  
cd ~
```
#8. Install Java: On the same VM, install Java:
```
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
openjdk version "17.0.13" 2024-10-15
OpenJDK Runtime Environment (build 17.0.13+11-Debian-2)
OpenJDK 64-Bit Server VM (build 17.0.13+11-Debian-2, mixed mode, sharing)
```

#9. Install Jenkins: 
```
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```
#10. Start Jenkins:
```
sudo systemctl start jenkins
sudo systemctl enable Jenkins
```
#11. Access Jenkins:
Open a browser and go to http://<public ip>:8080.
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
#12. Install Required Plugins:  
Go to "Manage Jenkins" > "Manage Plugins" > "Available".  
Search for and install:  
GitHub Integration Plugin: For GitHub connectivity.  
Pipeline Plugin: For defining the pipeline (optional, often pre-installed).  
Restart Jenkins if prompted.

#13. Step 3: Set Up GitHub Repository  
Create a Repository:  
On GitHub, create a new repository (e.g., my-static-website).  
Initialize it with a basic index.html:  
```
<!DOCTYPE html>
<html>
<body>
  <h1>My Static Website</h1>
  <p>Version 1.0</p>
</body>
</html>
```
Commit and push this file to the main branch.

#14. Generate SSH Keys for Jenkins:  
On the VM, generate an SSH key pair
```
ssh-keygen -t rsa -b 4096 -C "jenkins@azurevm" -f ~/.ssh/jenkins_key
```
Press Enter to accept defaults (no passphrase for simplicity).  
Add the public key (~/.ssh/jenkins_key.pub) to your GitHub repository:  
```
cat .ssh/jenkins_key.pub 
```
GitHub > Repository > Settings > Deploy keys > Add deploy key.  
Paste the public key and enable "Allow write access."  

#15. Configure Jenkins SSH:  
In Jenkins, go to "Manage Jenkins" > "Manage Credentials" > "System" > "Global credentials" > "Add Credentials".  
Set:
```  
Kind: SSH Username with private key  
Username: git  
Private Key: Paste the contents of ~/.ssh/jenkins_key (from the VM)  
ID: github-ssh-key  
Description: GitHub SSH Key  
Save. 
``` 

#16. Step 4: Create the Jenkins Pipeline
Create a New Pipeline Job:  
In Jenkins, click "New Item" > Name it deploy-static-website > Select "Pipeline" > Click "OK".  

#17. Configure the Pipeline:  
Under "General", check "GitHub project" and enter your repository URL (e.g., https://github.com/yourusername/my-static-website).  
Under "Build Triggers", check "GitHub hook trigger for GITScm polling".  
Under "Pipeline", set:  
Definition: Pipeline script
Script: Add the following:
```
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-ssh-key', url: 'https://github.com/AnandSShekhawat/my-static-website.git'
            }
        }
        stage('Deploy to VM') {
            steps {
                sh 'cp -r * /var/www/mywebsite/'
                sh 'sudo systemctl restart nginx'
            }
        }
    }
}
```
build the pipeline.

#18. The build failed twice 
```
Error 1:
+ cp -r README.md index.html /var/www/mywebsite/
cp: cannot create regular file '/var/www/mywebsite/README.md': Permission denied
cp: cannot create regular file '/var/www/mywebsite/index.html': Permission denied
```

Explanation =  This error occurs because the Jenkins user doesn't have the necessary permissions to write files into /var/www/mywebsite/.  
Solution = 
Grant Jenkins User Permissions to /var/www/mywebsite/  
Run the following command:
```
ps aux | grep jenkins
```
Usually, Jenkins runs under the user jenkins.  
Change the ownership of /var/www/mywebsite/ to the Jenkins user:  
```
sudo chown -R jenkins:jenkins /var/www/mywebsite/
```
#Grant necessary permissions:
```
sudo chmod -R 755 /var/www/mywebsite/
```
Build now: failed
```
Error 2:
+ cp -r README.md index.html /var/www/mywebsite/
+ sudo systemctl restart nginx
sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
sudo: a password is required
```
Explanation:
This error happens because Jenkins is trying to use sudo in a non-interactive shell, and it requires a password.  
Here’s how you can fix it:

Solution:  
Allow Jenkins to Run sudo Without a Password
```
sudo visudo
Add the following line at the bottom (replace jenkins with your actual Jenkins user if it's different):
jenkins ALL=(ALL) NOPASSWD: /bin/cp, /bin/systemctl restart nginx
Save and exit (CTRL + X, then Y, then Enter).
sudo systemctl restart jenkins
```
Build now : Successfull

#19. GitHub Configuration:  
Go to your GitHub repository → Settings → Webhooks
Add webhook:  
Payload URL: http://your-vm-ip:8080/github-webhook/  
Content type: application/json  
Select "Push events"

#20. Clone your repo to the local machine, make changes,  
add and commit changes and push the changes to the repo,  
jenkins will automatically trigger the build and deploy the changes.  

## SUCCESS!! 




