# ActiveMQ Artemis

##########################################################
## Build Image                                           #
##########################################################
FROM openjdk:8 as builder
LABEL maintainer="Victor Romero <victor.romero@gmail.com>"

ARG ACTIVEMQ_ARTEMIS_VERSION

RUN apt-get -qq -o=Dpkg::Use-Pty=0 update && \
  apt-get -qq -o=Dpkg::Use-Pty=0 install -y --no-install-recommends \
    libaio1=0.3.110-3 \
    xmlstarlet=1.6.1-2 \
    jq=1.5+dfsg-1.3 \
    ca-certificates=20161130+nmu1 \
    wget=1.18-5+deb9u2 && \
  rm -rf /var/lib/apt/lists/*

# Uncompress and validate
WORKDIR /opt
RUN wget "https://repository.apache.org/content/repositories/releases/org/apache/activemq/apache-artemis/${ACTIVEMQ_ARTEMIS_VERSION}/apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}-bin.tar.gz" && \
  wget "https://repository.apache.org/content/repositories/releases/org/apache/activemq/apache-artemis/${ACTIVEMQ_ARTEMIS_VERSION}/apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}-bin.tar.gz.asc" && \
  wget "http://apache.org/dist/activemq/KEYS" && \
  gpg --import "KEYS" && \
  gpg "apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}-bin.tar.gz.asc" && \
  tar xfz "apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}-bin.tar.gz" && \
  ln -s "/opt/apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}" "/opt/apache-artemis" && \
  rm -f "apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}-bin.tar.gz" "KEYS" "apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}-bin.tar.gz.asc"

# Create broker instance
# Per recommendation of https://activemq.apache.org/artemis/docs/latest/perf-tuning.html : -XX:+AggressiveOpts -XX:+UseFastAccessorMethods -XX:+UseParallelOldGC 
WORKDIR /var/lib
RUN if test "${ACTIVEMQ_ARTEMIS_VERSION}" = "1.0.0" ; \
    then \
      echo n | "/opt/apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}/bin/artemis" create artemis \
        --home /opt/apache-artemis \
        --user artemis \
        --password simetraehcapa \
        --cluster-user artemisCluster \
        --cluster-password simetraehcaparetsulc ; \
    else \
      "/opt/apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}/bin/artemis" create artemis \
        --home /opt/apache-artemis \
        --user artemis \
        --password simetraehcapa \
        --role amq \
        --require-login \
        --cluster-user artemisCluster \
        --cluster-password simetraehcaparetsulc ; \
    fi 

WORKDIR /var/lib/artemis/etc

# --java-options doesn't seem to work across the board on all versions adding them manually
RUN sed -i "s/JAVA_ARGS=\"/JAVA_ARGS=\"-Djava.net.preferIPv4Addresses=true -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=2 /g" ../etc/artemis.profile

# Ports are only exposed with an explicit argument, there is no need to binding
# the web console to localhost
RUN xmlstarlet ed -L -N amq="http://activemq.org/schema" \
    -u "/amq:broker/amq:web/@bind" \
    -v "http://0.0.0.0:8161" bootstrap.xml

# Remove default values for memory in artemis profile in order to let the automatic
# Java ergonomics detection work https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html
RUN sed -i "s/-Xm[xs][^ \"]*//g" ../etc/artemis.profile

# For the casual run of the image make the docker-entrypoint-sh think 
# that the performance journal calibration is already completed
RUN if (echo "${ACTIVEMQ_ARTEMIS_VERSION}" | grep -Eq  "(1.5\\.[3-5]|[^1]\\.[0-9]\\.[0-9]+)" ) ; then touch /var/lib/artemis/data/.perf-journal-completed; fi

##########################################################
## Run Image                                             #
##########################################################
FROM openjdk:8
LABEL maintainer="Victor Romero <victor.romero@gmail.com>"
ARG ACTIVEMQ_ARTEMIS_VERSION
ENV ACTIVEMQ_ARTEMIS_VERSION=$ACTIVEMQ_ARTEMIS_VERSION

# add user and group for artemis
RUN groupadd -r artemis && useradd -r -g artemis artemis

RUN apt-get -qq -o=Dpkg::Use-Pty=0 update && \
  apt-get -qq -o=Dpkg::Use-Pty=0 install -y --no-install-recommends \
    libaio1=0.3.110-3 \
    xmlstarlet=1.6.1-2 \
    jq=1.5+dfsg-1.3 \
    dumb-init=1.2.0-1 \ 
    gwenhywfar-tools=4.15.3-5+b1 && \
  rm -rf /var/lib/apt/lists/*

COPY --from=builder "/opt/apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}" "/opt/apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}"
COPY --from=builder "/var/lib/artemis" "/var/lib/artemis"

RUN ln -s "/opt/apache-artemis-${ACTIVEMQ_ARTEMIS_VERSION}" /opt/apache-artemis && chmod 755 /opt/apache-artemis
COPY assets/etc/* /var/lib/artemis/etc/
RUN chown -R artemis.artemis /var/lib/artemis

RUN mkdir -p /opt/assets
COPY assets/enable-jmx.xml /opt/assets

# Web Server
EXPOSE 8161

# Port for CORE,MQTT,AMQP,HORNETQ,STOMP,OPENWIRE
EXPOSE 61616

# Port for HORNETQ,STOMP
EXPOSE 5445

# Port for AMQP
EXPOSE 5672

# Port for MQTT
EXPOSE 1883

#Port for STOMP
EXPOSE 61613

# Expose some outstanding folders
USER root
#VOLUME ["/var/lib/artemis/data"]
#VOLUME ["/var/lib/artemis/tmp"]
#VOLUME ["/var/lib/artemis/etc"]
#VOLUME ["/var/lib/artemis/etc-override"]

RUN chgrp -R 0 /var/lib/artemis
RUN chmod -R g=u /var/lib/artemis
WORKDIR /var/lib/artemis/bin


COPY assets/docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["artemis-server"]
