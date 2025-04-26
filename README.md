# Deploy Dockerized Node.js App with MySQL in Custom AWS VPC

This guide explains how to deploy a Dockerized Node.js application with a MySQL database into a custom AWS Virtual Private Cloud (VPC). The application will run in a **public subnet** and the MySQL database in a **private subnet**, while allowing secure communication between them.

---

## 🏗️ Step 1: Create a Custom VPC

### Configuration:
- **VPC CIDR block**: `10.0.0.0/16`
- **Public Subnet**: `10.0.1.0/24`
- **Private Subnet**: `10.0.2.0/24`

### Instructions:
1. Go to AWS VPC Dashboard.
2. Click on **Create VPC** > Select **VPC only**.
3. Assign CIDR block `10.0.0.0/16`.
4. Create two subnets under this VPC:
   - Public Subnet: `10.0.1.0/24` (Enable Auto-assign Public IP)
   - Private Subnet: `10.0.2.0/24`
5. Create and attach an **Internet Gateway** to the VPC.
6. Create a **Route Table** for the public subnet and add a route to the internet (0.0.0.0/0 via IGW).
7. Associate the public subnet with this route table.
8. For internet access in the private subnet, create a **NAT Gateway** in the public subnet, allocate an Elastic IP, and create a route in the private subnet’s route table to the internet via the NAT Gateway.

---

## 🔐 Step 2: Configure Security Groups

### App Security Group (Public EC2):
- Inbound: TCP 3000 from your IP or anywhere (`0.0.0.0/0`)
- Outbound: All traffic

### DB Security Group (Private EC2):
- Inbound: TCP 3306 from App Security Group
- Outbound: All traffic

---

## 💻 Step 3: Launch EC2 Instances

### Public EC2 Instance (App):
- AMI: Ubuntu Server 22.04
- Subnet: Public Subnet
- Assign Public IP
- Attach App Security Group

### Private EC2 Instance (DB):
- AMI: Ubuntu Server 22.04
- Subnet: Private Subnet
- No Public IP
- Attach DB Security Group

---

## 🐳 Step 4: Install Docker on Both EC2s
SSH into each instance and run:
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```
*NB:* You have to SSH into the private instance from the public instance.
Copy the key pair of the private instance from the local machine to the public instance using the scp command
---

## 📦 Step 5: Setup Dockerized App and Database

### Dockerfile (Node.js App):
```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --production
COPY . .
CMD ["node", "src/index.js"]
EXPOSE 3000
```

### docker-compose.yml (Only for Reference):
```yaml
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

---

## 🚀 Step 6: Deploy Containers

### On Private EC2 (MySQL):
```bash
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  -v todo-mysql-data:/var/lib/mysql \
  -p 3060:3060 \
  mysql:8.0
 
```

*OR*

*Create a Docker-Compose file for the MySQL DB*
```yaml
services:
  mysql:
    image: mysql:8.0
    ports:
      - 3306:3306
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

### On Public EC2 (App):
Update `MYSQL_HOST` to use the **Private IP** of the Private EC2 (DB):
```bash
docker build -t my-node-app .
docker run -d \
  -p 3000:3000 \
  -e MYSQL_HOST=<PRIVATE_EC2_PRIVATE_IP> \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  my-node-app
```

### OR

```yaml
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: private_instance_ip
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```
---

## ✅ Step 7: Test the Setup

- Access the app at: `http://<PUBLIC_EC2_PUBLIC_IP>:3000`
- Check logs: `docker logs <container_id>` to ensure DB connection works.

---

## 🔗 Resources:
- App Repo: https://github.com/docker/getting-started-app
- AWS VPC: https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html
- Docker: https://docs.docker.com/
