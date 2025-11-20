# Étape 1 - Build Maven
FROM maven:3.9-alpine AS builder

# Dossier de travail dans le conteneur
WORKDIR /app

# Copie du pom.xml en premier
COPY pom.xml .

# Copie du code source
COPY src ./src

RUN mvn clean package

# Étape 2 - Image finale légère
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]