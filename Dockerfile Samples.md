# Dockerfile Samples
## 1. Dockerfile (Node.js, Python, Java)
### A. Dockerfile – Node.js App
```
# Base image
FROM node:18

# Create app directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm install --only=production

# Copy application source
COPY . .

# Expose port
EXPOSE 3000

# Run Node app
CMD ["node", "app.js"]
```
### B. Dockerfile – Python App (Flask)
```
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```
### C. Dockerfile – Java Spring Boot App
```
FROM eclipse-temurin:17-jdk AS build

WORKDIR /app

COPY . .
RUN ./mvnw clean package -DskipTests

FROM eclipse-temurin:17-jre

WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```
### 1.2 Docker Compose (Web + DB)
#### Node.js + MySQL
```
version: "3.9"

services:
  web:
    image: node:18
    container_name: node_app
    working_dir: /app
    volumes:
      - ./app:/app
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASS: mypass
    command: ["npm", "start"]

  db:
    image: mysql:8
    container_name: mysql_db
    environment:
      MYSQL_ROOT_PASSWORD: mypass
      MYSQL_DATABASE: testdb
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```
### 1.3 Multi-Stage Build Example (Node.js)
#### Highly asked in interviews.
```
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app

COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Run
FROM node:18-slim
WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY package*.json ./

RUN npm install --only=production

EXPOSE 3000
CMD ["node", "dist/server.js"]
```
### 1.4 Private Registry Push/Pull
#### Push image
```
docker login myregistry.com
docker build -t myregistry.com/myapp:v1 .
docker push myregistry.com/myapp:v1
```
#### Pull image
```
docker pull myregistry.com/myapp:v1
```
### 1.5 Docker Networking
#### Create a custom network
```
docker network create my-net
```
#### Run containers inside network
```
docker run -d --name db --network my-net mysql:8
docker run -d --name app --network my-net node-app-image
```
### 1.6 Docker Volumes
#### Create & use volumes
```
docker volume create app-data
docker run -d -v app-data:/var/lib/mysql mysql:8
```
#### Bind mount example
```
docker run -d -v $(pwd)/logs:/app/logs node-app-image
```
