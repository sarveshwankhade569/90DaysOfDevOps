Launch EC2
   AMI: Ubuntu 22.04
   Instance type: t2.micro (free tier)
   Key pair: create/download .pem
   Security Group:
       SSH → Port 22
       HTTP → Port 80
Step 2: Connect via SSH
      chmod 400 your-key.pem
      ssh -i your-key.pem ubuntu@<your-instance-ip>
Part 2: Install Docker & Nginx
        Step 1: Update System
                sudo apt update && sudo apt upgrade -y
        Step 2: Install Docker
                sudo apt install docker.io -y
                sudo systemctl start docker
                sudo systemctl enable docker
                docker --version - To check docker version
        Step 3: Install Nginx
                sudo apt install nginx -y
                sudo systemctl start nginx
                sudo systemctl enable nginx
                systemctl status nginx - Verify Nginx
Part 3: Security Group
             | Type | Port | Source    |
             | ---- | ---- | --------- |
             | SSH  | 22   | My IP     |
             | HTTP | 80   | 0.0.0.0/0 |
       Test in Browser
            http://<your-instance-ip>
Part 4: Extract Nginx Logs
       Step 1: View Logs
               cd /var/log/nginx
               ls
               cat access.log
               cat error.log
       Step 2: Save Logs to File
               cat /var/log/nginx/access.log > ~/nginx-logs.txt
               cat /var/log/nginx/error.log >> ~/nginx-logs.txt - Append error logs
       Step 3: Download Logs to Local
               scp -i your-key.pem ubuntu@<your-instance-ip>:~/nginx-logs.txt . - Run this on your local machine

       PRACTICE TASK: 
                 Stop Nginx: sudo systemctl stop nginx
                 Check:      curl http://localhost
                 Fix:        sudo systemctl start nginx
       Task 2: Change Web Page
              sudo vim /var/www/html/index.nginx-debian.html

              Replace content: <h1>Hello from Sarvesh DevOps Server</h1>
                               sed -i 's/word1/word2/' file name :- to replace the word1 with word2
                               sed -i 's/word1/woed2/g' wile name :- to replace all word in the file
              Task 3: Check Logs in Real-Time
                      tail -f /var/log/nginx/access.log : Refresh browser and watch logs live
              Task 4: Run Nginx via Docker
                      sudo docker run -d -p 8080:80 nginx
                      http://<your-ip>:8080 : Now open
              Commands Used:
                       ssh -i key.pem ubuntu@ip
                       sudo apt update
                       sudo apt install nginx
                       systemctl start nginx
                       cat /var/log/nginx/access.log
                       scp ...
              
                
