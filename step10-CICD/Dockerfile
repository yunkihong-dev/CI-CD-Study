FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# 실행 이미지 (보안 패치 최신)
FROM openjdk:17-jdk-slim-bookworm

RUN apt-get update && apt-get upgrade -y && apt-get clean

ARG TEAM
ARG COLOR
ENV TEAM_NAME=${TEAM}
ENV TEAM_COLOR=${COLOR}
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
