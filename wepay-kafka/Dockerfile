FROM debezium/connect:1.7

ARG JMX_AGENT_VERSION
RUN mkdir /kafka/etc && cd /kafka/etc &&\
        curl -so jmx_prometheus_javaagent.jar \
        https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.15.0/jmx_prometheus_javaagent-0.15.0.jar

COPY config.yml /kafka/etc/config.yml
COPY wepay-kafka-connect-bigquery-2.2.2 /kafka/connect/wepay-kafka-connect-bigquery-2.2.2
COPY cdc-dev-credential.json /kafka/connect/cdc-dev-credential.json
COPY cdc-prod-credential.json /kafka/connect/cdc-prod-credential.json