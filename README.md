## Pontos bem básicos
-Builder: Para fazer o deploy ou eu crio em package, ou rodo o comando ./mvn package, onde gerará a pasta target .jar que será o arquivo de deploy.
- Deploy:
- Para rodar uma aplicação
```bash
java: java -jar "nome ou caminho do arquivo"
```
Receita básica para criar um ambiente java em docker file:
```dockerfile
FROM eclipse-temurin:17-jdk-alpine as builder
WORKDIR application
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src
RUN ./mvnw package -DskipTests
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract


FROM eclipse-temurin:17-jre-alpine
WORKDIR application
COPY --from=builder application/dependencies/ ./
COPY --from=builder application/spring-boot-loader/ ./
COPY --from=builder application/snapshot-dependencies/ ./
COPY --from=builder application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]

```

Depois, cria-se uma imagem na máquina:
```bash
`docker build -t matheus/codechella:1.0 .`
```

## Configuração do application-prod.properties:

```text
spring.datasource.url=${DATASOURCE_URL}
spring.datasource.username=${DATASOURCE_USERNAME}
spring.datasource.password=${DATASOURCE_PASSWORD}

spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=false

spring.mail.host=${MAIL_HOST}
spring.mail.username=${MAIL_USER}
spring.mail.password=${MAIL_PASSWORD}
spring.mail.port=587
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true

app.security.jwt.secret=${APP_JWT_SECRET}

```

Configuração básica do docker-compose:
```dockerfile

version: '3'
services:
  mysql:
    image: mysql:8.0.36
    env_file: ./env/mysql.env
    volumes:
      - ./mysql-data:/var/lib/mysql
    restart: unless-stopped
    healthcheck:
      test: mysqladmin ping -h 127.0.0.1 -u $$MYSQL_USER --password=$$MYSQL_PASSWORD
      interval: 5s
      timeout: 5s
      retries: 10

  app:
    build:
      context: .
    env_file: ./env/app.env
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy

volumes:
  mysql-data:

```

Como colocou os envs, deve-se criar dois para configuração, para variáveis de ambiente:
app.env:
```text
SPRING_PROFILES_ACTIVE=prod
SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/codechella
SPRING_DATASOURCE_USERNAME=codechella_user
SPRING_DATASOURCE_PASSWORD=codechella_pwd
MAIL_HOST=smtp.gmail.com
MAIL_USER=mail_user
MAIL_PASSWORD=mail_pwd
APP_JWT_SECRET=jwt_secret

```
mysql.env
```text
MYSQL_RANDOM_ROOT_PASSWORD=true
MYSQL_DATABASE=codechella
MYSQL_USER=codechella_user
MYSQL_PASSWORD=codechella_pwd

```


Depois rodas os container com o docker compose:

```bash
docker compose up --build
```

