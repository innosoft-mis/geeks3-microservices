# geeks2-microservices
GEEKS 2 : Microservices

## Lab 0 : ติดตั้ง Docker
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

## Lab 1 : docker image pull สำหรับ images ที่จำเป็น

ใช้คำสั่ง docker image pull สำหรับ images ที่จำเป็น
```sh
sudo docker image pull python:3.9.7
sudo docker image pull mariadb
sudo docker image pull phpmyadmin/phpmyadmin
sudo docker image pull portainer/portainer-ce:latest
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
สร้างไฟล์ my.conf ใน hospital/config
```sh
nano hospital/init/01.sql
```
โดยในไฟล์ 01.sql ให้มีเนื้อหาดังนี้
```sql
CREATE DATABASE IF NOT EXISTS `hospital`;
GRANT ALL ON `hospital`.* TO 'user'@'%’;
```
เมื่อสร้าง directory และไฟล์ต่าง ๆ ที่จำเป็นเรียบร้อย ให้ใช้คำสั่ง
```sh
sudo docker-compose up -d
```
แล้วตรวจสอบว่า container ของ database ถูกสร้างสำเร็จ
```sh
sudo docker ps
```
## Lab 4 : ติดตั้ง phpMyAdmin เพื่อจัดการ Database
เพิ่มเนื้อหาส่วนนี้ เข้าในในส่วน services: ของ docker-compose.yml
```yml
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin
    environment:
      PMA_ARBITRARY: 1
    restart: always
    ports:
      - 8080:80
    networks:
      - moph-network
```
เมื่อแก้ไข docker-compose.yml เรียบร้อย ให้ใช้คำสั่ง
```sh
sudo docker-compose up -d
```
ทดสอบดูว่า phpmyadmin ถูกสร้างสำเร็จ
```
sudo docker ps
```
เข้าใช้งาน phpMyAdmin ผ่าน Web brownser ที่
```
http://localhost:8080
```
เข้าใช้งาน hospital-db ให้ระบุดังนี้
```
Server: hospital-db
Username: user
Password: user
```
เมื่อเข้าไปแล้วให้เลือก databse ชื่อ hospital แล้วไปที่ tab ชื่อ Import แล้วทำการ Import ข้อมูลจากไฟล์ chospital.sql (ดาวน์โหลดได้จาก shared drive)

## Lab 5 : สร้าง Microservice แรก กันเถอะ! (hospital service)
สร้าง Dockerfile ใน hospital/ โดย
```sh
nano hospital/Dockerfile
```
โดยใน Dockerfile มีเนื้อหาดังนี้
```dockerfile
FROM python:3.9.7
WORKDIR /usr/src/app
RUN apt-get update && apt-get install -y \
	libsasl2-dev \
	python-dev \
	libldap2-dev \
	libssl-dev
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN groupadd -r python && useradd -r -g python python
RUN apt-get update \ 
	&& apt-get install -y apt-transport-https \
	&& apt-get install mariadb-client -y
USER python
CMD ["fastapi", "run", "main.py", "--port", "80"]

```
สร้าง requirements.txt ใน hospital/ โดย
```sh
nano hospital/requirements.txt
```
โดยในไฟล์ requirements.txt มีเนื้อหาดังนี้
```
fastapi
pandas
pymysql
sqlalchemy
requests
```
สร้าง main.py ใน hospital/ โดย
```sh
nano hospital/main.py
```
โดยในไฟล์ main.py มีเนื้อหาดังนี้
```python
from fastapi import FastAPI
import pandas as pd 
import sqlalchemy as sa

app = FastAPI()

@app.get("/")
async def root():

	conn_str = 'mysql+pymysql://user:user@hospital-db:3306/hospital'
	engine = sa.create_engine(conn_str)
	conn = engine.connect()
	hospital = pd.read_sql("chospital", conn)
	conn.close()

	return hospital.to_dict("records")
```
เมื่อเพิ่มไฟล์ทั้ง 3 เรียบร้อย ขั้นต่อไปจะต้องไปแก้ไข docker-compose.yml โดยเพิ่มส่วน service: ดังนี้
```yml
  hospital:
    container_name: hospital
    ports:
      - "3001:80"
    build:
      context: ./hospital
      dockerfile: Dockerfile
    depends_on:
      - hospital-db
    volumes:
      - ./hospital:/usr/src/app
    networks:
      - moph-network
```
หลักจากแก้ไข  docker-compose.yml เรียบร้อย ก็ทำการ run คำสั่ง 
```sh
sudo docker-compose up -d
```
ถ้า service ทำงานถูกต้องสมบูรณ์จะต้องเข้าดูข้อมูล chospital ผ่าน web browser ได้ที่
```
http://localhost:3001
```

## Lab 6 : สร้าง Microservice ที่สอง (ncdscreen service)
ขั้นตอน
1. สร้าง database สำหรับ ncdscreen (คล้าย lab 3) แต่เปลี่ยนจาก hospital เป็น ncdscreen
2. นำข้อมูลเข้า database โดยใช้ phpMyAdmin (ncd_screen.sql อยู่ใน shared drive)
3. สร้าง microservice ชื่อ ncdscreen (คล้าย lab 5) แต่เปลี่ยนจาก hospital เป็น ncdscreen

สำหรับท่านที่ทำ lab ไม่ทัน สามารถ Download ไฟล์ที่เป็นผลของ lab 6 ได้ [ที่นี่](https://github.com/innosoft-mis/geeks2-microservices-lab/raw/main/files/lab6.zip)

## Lab 7 : ncdscreen service ต้องการข้อมูลจาก hospital service

โจทย์คือ ต้องการสร้าง API ที่ URL http://localhost:3002/with-hospital สำหรับแสดงข้อมูล ncd_screen พร้อมด้วยชื่อสถานพยาบาล

แก้ไขไฟล์ ncdscreen/main.py เพื่อเพิ่ม /with-hospital 
```python
from fastapi import FastAPI
import pandas as pd 
import sqlalchemy as sa
import requests

app = FastAPI()

def get_all_ncd_screen_data():

	conn_str = 'mysql+pymysql://user:user@ncdscreen-db:3306/ncdscreen'
	engine = sa.create_engine(conn_str)
	conn = engine.connect()
	hospital = pd.read_sql("ncd_screen", conn)
	conn.close()

	return hospital

@app.get("/")
async def root():
	ncd_screen = get_all_ncd_screen_data()
	return ncd_screen.to_dict("records")

@app.get("/with-hospital/")
async def with_hospital():
	ncd_screen = get_all_ncd_screen_data()
	r = requests.get("http://hospital")
	data = r.json()
	hospital = pd.DataFrame(data)
	new_ncd = pd.merge(ncd_screen, hospital, on='hospcode', how='inner')
	return new_ncd.to_dict("records")
```
เมื่อแก้เสร็จแล้วใช้ 2 คำสั่งนี้ 
```
sudo docker-compose down
sudo docker-compose up -d --build
```

## Lab 8 : แก้ไขและ deploy เฉพาะบาง service
เพิ่ม code แสดง status ของระบบ ใน ncdscreen/main.py
```python
@app.get("/status/")
async def status():
	return {'status': 'Online'}
```
ทำการ deploy เฉพาะ ncdscreen service นี้โดย
```sh
sudo docker-compose rm -s ncdscreen
sudo docker-compose up -d --no-deps --build ncdscreen
```
ดูผลลัพธ์ของ function ใหม่ที่เพิ่มเข้าไปผ่าน web browser ที่
```
http://localhost:3002/status/
```

## Lab 9 : ติดตั้ง Portainer
เพิ่ม volume ชื่อ portainer_data ใน docker-compose.yml
```yml
volumes:
  portainer_data: {}
```
เพิ่ม portainer ในส่วน services: ของ docker-compose.yml
```yml
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - 8888:8000
      - 9999:9000
      - 9443:9443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
```
เมื่อแก้เสร็จแล้วใช้คำสั่งนี้ 
```
sudo docker-compose up -d
```
หากการติดตั้งสมบูรณ์จะสามารถเข้าใช้งาน portainer ได้ผ่าน web browser ที่
```
http://localhost:9999
หรือ
https://localhost:9443
```
