ARG DEBEZIUM_VERSION=1.9
FROM debezium/connect:$DEBEZIUM_VERSION
ENV KAFKA_CONNECT_JDBC_DIR=$KAFKA_CONNECT_PLUGINS_DIR/kafka-connect-jdbc
ENV LD_LIBRARY_PATH=/instant_client/

# These should point to the driver version to be used
# ENV MAVEN_DEP_DESTINATION=$KAFKA_HOME/libs \
#     ORACLE_JDBC_REPO="com/oracle/database/jdbc" \
#     ORACLE_JDBC_GROUP="ojdbc8" \
#     ORACLE_JDBC_VERSION="21.1.0.0" \
#     ORACLE_JDBC_MD5=ebb69c8de3a83c08846b3fd47836ff15

COPY plugins/confluentinc-kafka-connect-gcs-*/* $KAFKA_CONNECT_PLUGINS_DIR/confluentinc-kafka-connect-gcs/
COPY plugins/wepay-kafka-connect-bigquery-*/* $KAFKA_CONNECT_PLUGINS_DIR/wepay-kafka-connect-bigquery/
COPY plugins/confluentinc-connect-transforms-*/* $KAFKA_CONNECT_PLUGINS_DIR/confluentinc-connect-transforms/
COPY plugins/confluentinc-kafka-connect-protobuf-converter-*/* $KAFKA_CONNECT_PLUGINS_DIR/confluentinc-kafka-connect-protobuf-converter/
COPY plugins/confluentinc-kafka-connect-json-schema-converter-*/* $KAFKA_CONNECT_PLUGINS_DIR/confluentinc-kafka-connect-json-schema-converter/
COPY plugins/debezium-scripting-*.jar $KAFKA_CONNECT_PLUGINS_DIR/kafka-connect-other-smt/
COPY plugins/groovy*.jar /kafka/libs/
# COPY plugins/debezium-connector-oracle/* /kafka/libs/

# RUN docker-maven-download central "$ORACLE_JDBC_REPO" "$ORACLE_JDBC_GROUP" "$ORACLE_JDBC_VERSION" "$ORACLE_JDBC_MD5"
COPY plugins/oracle_instantclient/* $LD_LIBRARY_PATH
COPY plugins/oracle_instantclient/xstreams.jar /kafka/libs
COPY plugins/oracle_instantclient/ojdbc8.jar /kafka/libs

USER root
RUN chmod 777 -R $LD_LIBRARY_PATH && \
    chmod 777 -R $KAFKA_CONNECT_PLUGINS_DIR/confluentinc-kafka-connect-gcs/ && \
    chmod 777 -R $KAFKA_CONNECT_PLUGINS_DIR/wepay-kafka-connect-bigquery/ && \
    chmod 777 -R $KAFKA_CONNECT_PLUGINS_DIR/confluentinc-connect-transforms/ && \
    chmod 777 -R $KAFKA_CONNECT_PLUGINS_DIR/confluentinc-kafka-connect-protobuf-converter/ && \
    chmod 777 -R $KAFKA_CONNECT_PLUGINS_DIR/confluentinc-kafka-connect-json-schema-converter/ && \
    chmod 777 -R $KAFKA_CONNECT_PLUGINS_DIR/kafka-connect-other-smt/ && \
    chmod 777 -R $KAFKA_CONNECT_PLUGINS_DIR/kafka-connect-other-smt/ && \
    chmod 777 -R /kafka/libs/groovy*.jar

USER kafka