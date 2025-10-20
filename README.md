# docker
# Stage 1: get Amazon Corretto 11 JDK
FROM my-artifactory/amazon-corretto:11 AS corretto

# Stage 2: use Maven base image (with Java 21)
FROM my-artifactory/maven:3.9.9-jdk-21 AS build

# Remove old Java 21 (optional, if it conflicts)
# The exact path depends on the base image — for Debian/Ubuntu-based Maven images, it's often:
# /usr/local/openjdk-21
RUN rm -rf /usr/local/openjdk* || true

# Copy Java 11 from Corretto image
COPY --from=corretto /usr/lib/jvm/java-11-amazon-corretto /usr/lib/jvm/java-11-amazon-corretto

# Set JAVA_HOME and PATH to use Java 11
ENV JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto
ENV PATH="$JAVA_HOME/bin:${PATH}"

# (Optional) Verify Java version
RUN java -version

# Your build steps
WORKDIR /app
COPY . .
RUN mvn clean package


FROM my-artifactory/maven:3.9.9-amazoncorretto-11

# Create non-root user
ARG POD_USER=producer
RUN useradd -m -u 1000 ${POD_USER}

# Set working directory (still as root)
WORKDIR /home/${POD_USER}

# Copy project files
COPY pom.xml .
COPY src ./src
COPY settings.xml /usr/share/maven/conf/settings.xml

# Fix ownership (important!)
RUN chown -R ${POD_USER}:${POD_USER} /home/${POD_USER}

# Switch to non-root user
USER ${POD_USER}

# Build with Maven using your internal Artifactory
RUN mvn -s /usr/share/maven/conf/settings.xml -f pom.xml clean package
