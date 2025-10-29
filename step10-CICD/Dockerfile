# 멀티 스테이지 빌드 - 빌드 단계
FROM gradle:8.5-jdk17 AS builder

WORKDIR /app

# Gradle 캐싱 최적화를 위해 빌드 파일 먼저 복사
COPY build.gradle settings.gradle ./
COPY gradle ./gradle

# 의존성 다운로드 (캐시 레이어)
RUN gradle dependencies --no-daemon || true

# 소스 코드 복사 및 빌드
COPY src ./src
RUN gradle clean build -x test --no-daemon

# 실행 이미지 - OpenJDK 17
FROM openjdk:17-jdk-slim

# Build arguments (GitHub Actions에서 전달)
ARG TEAM
ARG COLOR

# Environment variables
ENV TEAM_NAME=${TEAM}
ENV TEAM_COLOR=${COLOR}

WORKDIR /app

# 빌드된 JAR 파일만 복사
COPY --from=builder /app/build/libs/*.jar app.jar

# 애플리케이션 포트
EXPOSE 8080

# 실행 명령
ENTRYPOINT ["java", "-jar", "app.jar"]
