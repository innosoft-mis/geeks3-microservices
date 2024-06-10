# geeks2-microservices-lab
GEEKS 2 : Microservices Lab

## Lab 1 : ติดตั้ง Docker
```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# ติดตั้ง
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# ทดสอบว่าการติดตั้งสำเร็จ
sudo docker run hello-world
```

## Lab 2 : ติดตั้ง Docker compose

```sh
# ดาวน์โหลด
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# ให้สิทธิ execute
sudo chmod +x /usr/local/bin/docker-compose

# ทดสอบว่าติดตั้งสำเร็จ
docker-compose --version
```

## Lab 3 : สร้าง Database สำหรับ Microservice แรกของท่าน
```sh
mkdir microservices
cd microservices
nano docker-compose.yml
```
เนื้อหาภายในไฟล์ docker-compose.yml
```yml
version: '3'
services:
  hospital-db:
    image: mariadb
    container_name: hospital-db
    environment:
      MYSQL_ROOT_PASSWORD: my_secret_password
      MYSQL_DATABASE: hospital
      MYSQL_USER: user
      MYSQL_PASSWORD: user
    ports:
      - "6001:3306"
    volumes:
      - hospital-dbdata:/var/lib/mysql
      - hospital-dblog:/var/log/mysql
      - ./hospital/config/my.conf:/etc/mysql/conf.d/config-file.cnf
      - ./hospital/init:/docker-entrypoint-initdb.d
    networks:
      - moph-network
volumes:
  hospital-dbdata: {}
  hospital-dblog: {}
networks:
  moph-network:
    driver: bridge
```
เมื่อ บันทึกไฟล์ docker-compose.yml เรียบร้อย ให้ใช้คำสั่งต่อไปนี้ เพื่อสร้าง directory ที่จำเป็น
```sh
mkdir hospital
mkdir hospital/config
mkdir hospital/init
```
สร้างไฟล์ my.conf ใน hospital/config
```sh
nano hospital/config/my.conf
```
โดยในไฟล์ my.conf ให้มีเนื้อหาดังนี้
```cnf
[mysqld]
bind-address            = 0.0.0.0
max_connections         = 505
max_user_connections    = 500
```

## Lab 4 : 
