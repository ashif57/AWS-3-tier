# AWS-3-tier

```
VPC (single)
 ├── Public Subnet
 │    └── Bastion / Web EC2 (Nginx)
 │
 ├── Private Subnet
 │    ├── App EC2 (Dockerized Flask)
 │    └── RDS MySQL
 │
GitHub Actions → DockerHub → Private EC2 pulls image
```

You already said you’ll **start/stop/terminate**, so **2 EC2s is fine**.

---

# 🧠 IMPORTANT CONCEPT (READ FIRST – INTERVIEW GOLD)

* **Public subnet** = internet access
* **Private subnet** = NO public IP
* **Private EC2 cannot talk to GitHub directly**
* So we use **pull-based Docker deployment**

This is **REAL-WORLD DevOps**, not a hack.

---

# 🟢 STEP 1: CREATE VPC

* Name: `three-tier-vpc`
* CIDR: `10.0.0.0/16`

✅ Free

---

# 🟢 STEP 2: CREATE SUBNETS

### Public Subnet

* Name: `public-subnet`
* CIDR: `10.0.1.0/24`
* Auto-assign public IP: **ENABLE**

### Private Subnet

* Name: `private-subnet`
* CIDR: `10.0.2.0/24`
* Auto-assign public IP: **DISABLE**

---

# 🟢 STEP 3: INTERNET GATEWAY + ROUTING

### Internet Gateway

* Create + attach to VPC

### Public Route Table

Add:

```
0.0.0.0/0 → Internet Gateway
```

Associate → **public-subnet**

### Private Route Table

* NO internet route
* ❌ NO NAT Gateway (costly)

---

# 🟢 STEP 4: SECURITY GROUPS

### Bastion / Web EC2 SG

Inbound:

| Type | Port | Source    |
| ---- | ---- | --------- |
| SSH  | 22   | Your IP   |
| HTTP | 80   | 0.0.0.0/0 |

---

### App EC2 SG (PRIVATE)

Inbound:

| Type  | Port | Source     |
| ----- | ---- | ---------- |
| SSH   | 22   | Bastion SG |
| Flask | 5000 | Bastion SG |

❌ No public access

---

### RDS SG

Inbound:

| Type  | Port | Source     |
| ----- | ---- | ---------- |
| MySQL | 3306 | App EC2 SG |

---

# 🟢 STEP 5: LAUNCH BASTION / WEB EC2 (PUBLIC)

* AMI: Amazon Linux 2
* Type: `t2.micro`
* Subnet: **public-subnet**
* Public IP: ENABLE
* SG: Bastion SG

---

# 🟢 STEP 6: LAUNCH APP EC2 (PRIVATE)

* AMI: Amazon Linux 2
* Type: `t2.micro`
* Subnet: **private-subnet**
* Public IP: ❌ DISABLE
* SG: App SG

✔️ This EC2 is **completely private**

---

# 🟢 STEP 7: SSH FLOW (VERY IMPORTANT)

You **CANNOT** SSH directly into private EC2.

### Correct Flow

```bash
ssh ec2-user@<BASTION_PUBLIC_IP>
ssh ec2-user@<PRIVATE_EC2_PRIVATE_IP>
```

This is **industry standard**.

---

# 🟢 STEP 8: INSTALL DOCKER ON APP EC2

(On private EC2)

```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
logout
```

Login again.

---

# 🟢 STEP 9: CREATE DOCKERIZED FLASK APP (LOCAL)

### app.py

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Private App Layer is Running!"

app.run(host="0.0.0.0", port=5000)
```

### Dockerfile

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY app.py .
RUN pip install flask
CMD ["python", "app.py"]
```

---

# 🟢 STEP 10: PUSH IMAGE TO DOCKERHUB (FROM LOCAL / BASTION)

```bash
docker build -t yourname/flask-private-app .
docker login
docker push yourname/flask-private-app
```

---

# 🟢 STEP 11: PULL IMAGE ON PRIVATE EC2 (KEY STEP)

Private EC2:

```bash
docker pull yourname/flask-private-app
docker run -d -p 5000:5000 yourname/flask-private-app
```

✅ App runs **inside private subnet**

---

# 🟢 STEP 12: CONFIGURE NGINX ON BASTION (WEB TIER)

On **public EC2**:

```bash
sudo yum install nginx -y
sudo nano /etc/nginx/nginx.conf
```

```nginx
location / {
    proxy_pass http://<PRIVATE_EC2_PRIVATE_IP>:5000;
}
```

Restart:

```bash
sudo systemctl restart nginx
```

---

# 🟢 STEP 13: CREATE RDS (PRIVATE)

* Engine: MySQL
* Instance: `db.t3.micro`
* Public access: ❌ No
* Subnet: private
* SG: RDS SG

---

# 🟢 STEP 14: CONNECT APP → RDS

Install in container:

```bash
pip install pymysql
```

Use RDS endpoint (private).

---

# 🟢 STEP 15: CI/CD WITH GITHUB ACTIONS (SECURE WAY)

### Flow (IMPORTANT)

```
GitHub Actions
 → Build Docker image
 → Trivy scan
 → Push to DockerHub
 → Private EC2 pulls image
```

📌 GitHub **never touches private AWS**

This is **SECURE & PROFESSIONAL**.

---

# 🟢 STEP 16: STOP / TERMINATE (COST CONTROL)

After work:

* Stop both EC2s
* Stop or delete RDS
* Delete unused EBS

---


