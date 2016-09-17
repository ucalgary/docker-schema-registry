FROM openjdk:8u92-jre-alpine

MAINTAINER King Chung Huang <kchuang@ucalgary.ca>

ENV MAVEN_MAJOR=3 \
    MAVEN_VERSION=3.3.9 \
    SCHEMA_REGISTRY_VERSION=3.0.1 \
    SCHEMA_REGISTRY_HOME=/schema-registry \
    REST_UTILS_VERSION=3.0.1 \
    COMMON_VERSION=3.1.0

ARG MAVEN_OPTIONS="-DskipTests"

# Install required packages
RUN apk add --no-cache \
		bash

# Download and build Schema Registry
RUN log () { echo -e "\033[01;95m$@\033[0m"; } && \

	apk add --no-cache --virtual .fetch-deps \
		ca-certificates \
		openssl \
		tar && \

	apk add --no-cache --virtual .build-deps \
		openjdk8="$JAVA_ALPINE_VERSION" \
		rsync && \

	BUILD_DIR="$(mktemp -d)" && \

	log "Download and unpack Apache Maven" && \
	wget -O $BUILD_DIR/apache-maven-bin.tar.gz "https://www.apache.org/dist/maven/maven-$MAVEN_MAJOR/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz" && \
	wget -O $BUILD_DIR/apache-maven-bin.tar.gz.sha1 "https://www.apache.org/dist/maven/maven-$MAVEN_MAJOR/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz.sha1" && \
	echo "$(cat $BUILD_DIR/apache-maven-bin.tar.gz.sha1) *$BUILD_DIR/apache-maven-bin.tar.gz" | sha1sum -c - && \
	tar -xzf $BUILD_DIR/apache-maven-bin.tar.gz --directory=$BUILD_DIR && \

	log "Download, unpack, and build confluentinc/schema-registry" && \
	wget -O $BUILD_DIR/schema-registry.tar.gz "https://github.com/confluentinc/schema-registry/archive/v$SCHEMA_REGISTRY_VERSION.tar.gz" && \
	tar -xzf $BUILD_DIR/schema-registry.tar.gz --directory=$BUILD_DIR && \
	cd $BUILD_DIR/schema-registry-$SCHEMA_REGISTRY_VERSION && \
	$BUILD_DIR/apache-maven-$MAVEN_VERSION/bin/mvn $MAVEN_OPTIONS --file=$BUILD_DIR/schema-registry-$SCHEMA_REGISTRY_VERSION/pom.xml package && \

	log "Download, unpack, and build confluentinc/rest-utils" && \
	wget -O $BUILD_DIR/rest-utils.tar.gz "https://github.com/confluentinc/rest-utils/archive/v$REST_UTILS_VERSION.tar.gz" && \
	tar -xzf $BUILD_DIR/rest-utils.tar.gz --directory=$BUILD_DIR && \
	cd $BUILD_DIR/rest-utils-$REST_UTILS_VERSION && \
	$BUILD_DIR/apache-maven-$MAVEN_VERSION/bin/mvn $MAVEN_OPTIONS --file=$BUILD_DIR/rest-utils-$REST_UTILS_VERSION/pom.xml package && \

	log "Download, unpack, and build confluentinc/common" && \
	wget -O $BUILD_DIR/common.tar.gz "https://github.com/confluentinc/common/archive/common-$COMMON_VERSION.tar.gz" && \
	tar -xzf $BUILD_DIR/common.tar.gz --directory=$BUILD_DIR && \
	cd $BUILD_DIR/common-common-$COMMON_VERSION && \
	$BUILD_DIR/apache-maven-$MAVEN_VERSION/bin/mvn $MAVEN_OPTIONS --file=$BUILD_DIR/common-common-$COMMON_VERSION/pom.xml package && \

	log "Install the build packages" && \
	cp -r $BUILD_DIR/schema-registry-$SCHEMA_REGISTRY_VERSION/package-schema-registry/target/kafka-schema-registry-package-$SCHEMA_REGISTRY_VERSION-package $SCHEMA_REGISTRY_HOME && \
	rsync -a $BUILD_DIR/rest-utils-$REST_UTILS_VERSION/package/target/rest-utils-package-$REST_UTILS_VERSION-package/ $SCHEMA_REGISTRY_HOME && \
	rsync -a $BUILD_DIR/common-common-$COMMON_VERSION/package/target/common-package-$COMMON_VERSION-SNAPSHOT-package/ $SCHEMA_REGISTRY_HOME && \

	log "Clean up" && \
	rm -r "$BUILD_DIR" && \
	apk del .fetch-deps .build-deps

# Adjust the default schema-registry properties to connect to Zookeeper at zookeeper:2181
ENV ZOOKEEPER_HOST=zookeeper \
    ZOOKEEPER_PORT=2181 \
    KAFKA_HOST=kafka \
    KAFKA_PORT=9092
RUN sed -i "s/kafkastore.connection.url=.*/kafkastore.connection.url=$ZOOKEEPER_HOST:$ZOOKEEPER_PORT/g" $SCHEMA_REGISTRY_HOME/etc/schema-registry/schema-registry.properties

# Add wait-for-it script, for use in waiting for Zookeeper and Kafka
ADD https://raw.githubusercontent.com/ucalgary/wait-for-it/master/wait-for-it.sh /usr/local/bin/wait-for-it
RUN chmod 755 /usr/local/bin/wait-for-it

# Copy custom schema-registry-start script
COPY bin/schema-registry-docker-start $SCHEMA_REGISTRY_HOME/bin/schema-registry-docker-start

WORKDIR $SCHEMA_REGISTRY_HOME

ENV PATH=$PATH:$SCHEMA_REGISTRY_HOME/bin

EXPOSE 9092

CMD ["schema-registry-docker-start", "/schema-registry/etc/schema-registry/schema-registry.properties"]
