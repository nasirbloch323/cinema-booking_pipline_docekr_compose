# Cinema Booking Pipeline — Setup Guide

Ye guide is project ko ek fresh Ubuntu (EC2) machine par set up karne, build karne, aur Jenkins ke through deploy karne ke poore steps batati hai.

## Tech Stack

- Spring Boot (Java 21 — Eclipse Temurin JDK 21 Alpine)
- PostgreSQL 16
- RabbitMQ 3.13 (management UI)
- pgAdmin 4
- Docker & Docker Compose
- Jenkins (CI/CD)

---

## 1. Repository Clone Karo

```bash
git clone https://github.com/nasirbloch323/cinema-booking_pipline_docekr_compose.git
cd cinema-booking_pipline_docekr_compose
```

---

## 2. Docker Install Karo

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker
```

Verify:

```bash
docker --version
docker compose version
```

---

## 3. Jenkins Install Karo

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk-headless

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Jenkins UI: `http://<EC2-Public-IP>:8080`

### Jenkins ko Docker access do

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## 4. JDK Versions Clean Karo (agar multiple installed hain)

Agar Ubuntu host par JDK 17/19 jaise extra versions installed hain aur RAM/disk full ho rahi hai, sirf JDK 21 rakho:

```bash
dpkg --list | grep -i jdk
sudo apt remove --purge -y openjdk-17-jdk openjdk-17-jdk-headless openjdk-17-jre openjdk-17-jre-headless
sudo apt autoremove --purge -y
sudo apt clean
```

App container ke andar `eclipse-temurin:21-jdk-alpine` (slim equivalent) use hota hai — `Dockerfile` mein already set hai.

---

## 5. Maven Wrapper Generate Karo

Agar project mein `mvnw` / `.mvn` files missing hain:

```bash
mvn -N wrapper:wrapper -Dmaven=3.9.6
```

Ye `mvnw`, `mvnw.cmd`, aur `.mvn/` folder generate karega — inhe repo mein commit aur push karna zaroori hai (`.gitignore` mein inhe ignore mat karo).

```bash
git add -f mvnw mvnw.cmd .mvn/
git commit -m "Add maven wrapper files"
git push origin main
```

---

## 6. Docker Compose Port Configuration

Jenkins already host par port `8080` use karta hai, isliye `cinema-app` ka host port `8081` set kiya gaya hai (`docker-compose.yml`):

```yaml
cinema-app:
  ports:
    - "8081:8080"
```

EC2 Security Group mein ye ports allow karo:

| Port | Service |
|---|---|
| 8080 | Jenkins |
| 8081 | cinema-app |
| 5050 | pgAdmin |
| 15672 | RabbitMQ Management UI |
| 5432 | PostgreSQL (optional, restrict to own IP) |
| 5672 | RabbitMQ (optional, restrict to own IP) |

---

## 7. Application Build & Run

```bash
docker compose build cinema-app
docker compose up -d
```

Status check:

```bash
docker compose ps
docker compose logs -f cinema-app
```

Health check:

```bash
curl http://localhost:8081/actuator/health
```

---

## 8. Jenkins Pipeline

Project root mein `Jenkinsfile` maujood hai jo ye stages perform karta hai:

1. Git Checkout
2. Maven Build & Test
3. Docker Image Build
4. Stop Old Containers
5. Docker Compose Deploy
6. Health Check

Jenkins job mein **Pipeline script from SCM** select karke is repo ka URL aur `Jenkinsfile` path daal do.

---

## Notes

- `eclipse-temurin` use karo, `openjdk` Docker image **deprecated** ho chuki hai (`openjdk:21-jdk-slim` tag ab exist nahi karta).
- Alpine-based images halki hoti hain lekin `musl libc` use karti hain — agar native dependency issues aayein to `eclipse-temurin:21-jdk-jammy` (Ubuntu based) par switch kar lo.
- Jenkins ko docker commands chalane ke liye `docker` group membership zaroori hai, warna `permission denied` error aata hai.
