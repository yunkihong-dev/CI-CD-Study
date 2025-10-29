# 멀티 스테이지 빌드 - 빌드 단계
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# Maven 캐싱 최적화를 위해 pom.xml 먼저 복사
COPY pom.xml .
RUN mvn dependency:go-offline

# 소스 코드 복사 및 빌드
COPY src ./src
RUN mvn clean package -DskipTests

# 실행 이미지 - Eclipse Temurin Alpine (취약점 최소화)
FROM eclipse-temurin:17-jre-alpine

# 패키지 업데이트로 취약점 제거
RUN apk update && apk upgrade && rm -rf /var/cache/apk/*

# Build arguments (GitHub Actions에서 전달)
ARG TEAM
ARG COLOR

# Environment variables
ENV TEAM_NAME=${TEAM}
ENV TEAM_COLOR=${COLOR}

WORKDIR /app

# 빌드된 JAR 파일만 복사
COPY --from=builder /app/target/*.jar app.jar

# 애플리케이션 포트
EXPOSE 8080

# 실행 명령
ENTRYPOINT ["java", "-jar", "app.jar"]
