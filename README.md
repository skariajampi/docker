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

FROM my-artifactory/maven:3.9.9-amazoncorretto-11

# Create non-root user
ARG POD_USER=producer
RUN useradd -m -u 1000 ${POD_USER}

# Set working directory
WORKDIR /home/${POD_USER}

# Copy your Maven settings
COPY settings.xml /usr/share/maven/conf/settings.xml

# Copy your project files
COPY pom.xml .
COPY src ./src

# Ensure Maven local repo exists and is writable
RUN mkdir -p /home/${POD_USER}/.m2 && \
    chown -R ${POD_USER}:${POD_USER} /home/${POD_USER}

# Switch to non-root user
USER ${POD_USER}

# Override the default Maven config directory
ENV MAVEN_CONFIG=/home/${POD_USER}/.m2

# Now run Maven using your Artifactory settings
RUN mvn -s /usr/share/maven/conf/settings.xml -f pom.xml clean package
-------------------

🧩 Root Cause
When you replace Java, you effectively replace the JDK truststore (where Java stores CA certificates).
So even if your base image had /etc/ssl/certs or /usr/local/share/ca-certificates configured, your new Java 11 truststore doesn’t know about them.
Maven uses the Java truststore ($JAVA_HOME/lib/security/cacerts) when connecting via HTTPS — not the system one.
Hence, SSL fails for Artifactory.
✅ Solutions
Option 1: Copy the trusted certs into the new Java truststore
If your base image already had the right internal CA certs configured in Java 21, copy them to Corretto 11.
Steps:
Find the truststore path in your base image:
echo $JAVA_HOME/lib/security/cacerts
Export certs before replacing Java:
keytool -list -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit > /tmp/certs.txt
You can also export specific certs:
keytool -export -alias your-internal-ca -keystore $JAVA_HOME/lib/security/cacerts \
    -storepass changeit -rfc -file /tmp/internal-ca.pem
After switching to Corretto 11, import those certs:
keytool -import -trustcacerts -alias your-internal-ca \
    -file /tmp/internal-ca.pem \
    -keystore $JAVA_HOME/lib/security/cacerts \
    -storepass changeit -noprompt
Verify:
keytool -list -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit | grep your-internal-ca
Option 2: Use system certificates and reconfigure Corretto to trust them
Corretto (like most OpenJDK builds) may use a private truststore instead of system certs.
You can link or rebuild the truststore with system CAs.
In Docker:
FROM amazoncorretto:11

# Copy system certs into java truststore
RUN /usr/bin/keytool -importkeystore \
    -srckeystore /etc/pki/java/cacerts \
    -destkeystore $JAVA_HOME/lib/security/cacerts \
    -srcstorepass changeit -deststorepass changeit
or on Debian/Ubuntu-based systems:
RUN update-ca-certificates \
 && cp /etc/ssl/certs/java/cacerts $JAVA_HOME/lib/security/cacerts
Make sure /etc/ssl/certs contains your internal CA (added with update-ca-certificates or trust anchor).
Option 3: Don’t replace Java; just install Corretto alongside
If possible, don’t overwrite the existing JDK. Instead, install Corretto separately and configure Maven to use it:
FROM your-base-image-with-java21 AS builder

RUN yum install -y java-11-amazon-corretto

ENV JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto
ENV PATH=$JAVA_HOME/bin:$PATH
This way, all existing SSL/truststore settings remain intact.
🧠 Pro Tip
You can test Java SSL directly before Maven:
echo | openssl s_client -connect artifactory.mycompany.com:443
If that works but mvn clean package still fails, it’s definitely Java truststore.


# Force Maven to detect OS properties before going offline
RUN mvn -B help:evaluate -Dexpression=os.detected.classifier -q -DforceStdout

# Now safely go offline
RUN mvn -B dependency:go-offline
