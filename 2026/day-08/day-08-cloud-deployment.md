
## Part 1: Launch EC2 Instance

| Setting        | Value                          |
|----------------|--------------------------------|
| AMI            | Ubuntu 22.04                   |
| Instance type  | t2.micro (Free Tier)           |
| Key pair       | Create/download a new `.pem`   |
| Security Group | SSH (22), HTTP (80)            |

### Security Group Rules

| Type | Port | Source      |
|------|------|-------------|
| SSH  | 22   | My IP       |
| HTTP | 80   | 0.0.0.0/0   |

### Connect via SSH
```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<your-instance-ip>
```

---

## Part 2: Install Docker & Nginx

### Step 1 — Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2 — Install Docker
```bash
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version   # check docker version
```

### Step 3 — Install Nginx
```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
systemctl status nginx   # verify nginx is running
```

### Test in Browser
```
http://<your-instance-ip>
```

---

## Part 3: Extract Nginx Logs

### Step 1 — View logs
```bash
cd /var/log/nginx
ls
cat access.log
cat error.log
```

### Step 2 — Save logs to a file
```bash
cat /var/log/nginx/access.log > ~/nginx-logs.txt
cat /var/log/nginx/error.log >> ~/nginx-logs.txt   # append error logs
```

### Step 3 — Download logs to local machine
Run this on your **local machine** (not the server):
```bash
scp -i your-key.pem ubuntu@<your-instance-ip>:~/nginx-logs.txt .
```

---

## Part 4: Practice Tasks

### Task 1 — Stop / Check / Fix Nginx
```bash
sudo systemctl stop nginx     # stop
curl http://localhost         # check (should fail)
sudo systemctl start nginx    # fix
```

### Task 2 — Change the Web Page
```bash
sudo vim /var/www/html/index.nginx-debian.html
```
Replace the content with:
```html
<h1>Hello from Sarvesh DevOps Server</h1>
```

**Useful `sed` replacements:**
```bash
sed -i 's/word1/word2/' filename      # replace first match per line
sed -i 's/word1/word2/g' filename     # replace all matches in the file
```

### Task 3 — Watch Logs in Real-Time
```bash
tail -f /var/log/nginx/access.log
```
Refresh the browser and watch requests appear live.

### Task 4 — Run Nginx via Docker
```bash
sudo docker run -d -p 8080:80 nginx
```
Open:
```
http://<your-ip>:8080
```

---

## Quick Command Reference

| Command | Purpose |
|---|---|
| `ssh -i key.pem ubuntu@ip` | Connect to EC2 instance |
| `sudo apt update` | Refresh package lists |
| `sudo apt install nginx` | Install Nginx |
| `systemctl start nginx` | Start Nginx |
| `cat /var/log/nginx/access.log` | View access logs |
| `scp -i key.pem ubuntu@ip:~/file .` | Copy file from server to local |

---

