# =============================================================================
# Multi-stage Dockerfile — Order Service
# Stage 1: Build the fat JAR with Maven
# Stage 2: Minimal JRE runtime image
# =============================================================================

# ---- Stage 1: Build ----
FROM maven:3.9.6-eclipse-temurin-21 AS build

WORKDIR /app

# Copy POM first so layer cache is reused if only sources change
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build, skipping tests (tests run in CI before Docker build)
COPY src ./src
RUN mvn package -DskipTests -B

# ---- Stage 2: Runtime ----
FROM eclipse-temurin:21-jre-alpine AS runtime

# Install curl for health-check probe
RUN apk add --no-cache curl

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

WORKDIR /app

# Copy only the fat JAR from the build stage
COPY --from=build /app/target/order-service-*.jar app.jar

EXPOSE 8080

# Use exec form so signals propagate correctly (graceful shutdown)
ENTRYPOINT ["java", \
            "-XX:+UseContainerSupport", \
            "-XX:MaxRAMPercentage=75.0", \
            "-XX:+UseG1GC", \
            "-Djava.security.egd=file:/dev/./urandom", \
            "-jar", "app.jar"]
