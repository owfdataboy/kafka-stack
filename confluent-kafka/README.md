# Kafka Connect: CDC from Oracle to GCS

# Setting up Oracle

1. Create a bridge network:
    
    ```bash
    docker network create dev_net --driver=bridge
    ```
    
2. Spin up an Oracle instance:
    
    ```bash
    # Create an Oracle container
    docker compose up -f ./oracle.docker-compose.yml up -d
    
    # View Oracle container logs
    docker logs -f oracle-12-ee
    ```
    
    Patiently wait for about 5-10 minutes until you see `Done with scripts we are ready to go` line the log console.
    
3. Try to connect to the Oracle using the following credential to verify that your database is up and running:
    
    ```
    Host: localhost
    Port: 1521
    SID: ORCLCDB
    Username: sys
    Role: sysdba
    Password: password
    ```
    
4. To enable XStream, you first has to access to your Oracle container using the following command:
    
    ```bash
    docker exec -it oracle-12-ee bash
    ```
    
    Then create a new directory for archive log:
    
    ```bash
    mkdir -p /opt/oracle/oradata/recovery_area
    chmod 777 -R /opt/oracle/oradata/recovery_area
    ```
    
    Then connect to the Oracle using sqlplus:
    
    ```bash
    export ORACLE_SID=ORCLCDB && sqlplus /nolog
    
    CONNECT sys/password AS SYSDBA
    ```
    
    Set up archive log and enable archive mode:
    
    ```sql
    alter system set db_recovery_file_dest_size = 5G;
    alter system set db_recovery_file_dest = '/opt/oracle/oradata/recovery_area' scope=spfile;
    alter system set enable_goldengate_replication=true;
    shutdown immediate
    startup mount
    alter database archivelog;
    alter database open;
    ```
    
    Make sure the archive mode has been enabled:
    
    ```sql
    archive log list;
    ```
    
    Enable XStream API ([reference](https://debezium.io/documentation/reference/1.9/connectors/oracle.html#_creating_xstream_users_for_the_connector))
    
    ```bash
    sqlplus sys/password@//localhost:1521/ORCLCDB as sysdba
      
    CREATE TABLESPACE xstream_adm_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/xstream_adm_tbs.dbf'
        SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
    
    CREATE USER dbzadmin IDENTIFIED BY dbz DEFAULT TABLESPACE xstream_adm_tbs QUOTA UNLIMITED ON xstream_adm_tbs;
    
    GRANT CREATE SESSION, SET CONTAINER TO dbzadmin;
    
    BEGIN
       DBMS_XSTREAM_AUTH.GRANT_ADMIN_PRIVILEGE(
          grantee                 => 'dbzadmin',
          privilege_type          => 'CAPTURE',
          grant_select_privileges => TRUE
       );
    END;
    /
    
    CREATE TABLESPACE xstream_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/xstream_tbs.dbf'
        SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
    
    CREATE USER dbzuser IDENTIFIED BY dbz
        DEFAULT TABLESPACE xstream_tbs
        QUOTA UNLIMITED ON xstream_tbs;
    
    GRANT CREATE SESSION TO dbzuser;
    GRANT SET CONTAINER TO dbzuser;
    GRANT SELECT ON V_$DATABASE to dbzuser;
    GRANT FLASHBACK ANY TABLE TO dbzuser;
    GRANT SELECT_CATALOG_ROLE TO dbzuser;
    GRANT EXECUTE_CATALOG_ROLE TO dbzuser;
    
    DECLARE
      tables  DBMS_UTILITY.UNCL_ARRAY;
      schemas DBMS_UTILITY.UNCL_ARRAY;
    BEGIN
        tables(1)  := NULL;
        schemas(1) := 'dbzuser';
      DBMS_XSTREAM_ADM.CREATE_OUTBOUND(
        server_name     =>  'dbzxout',
        table_names     =>  tables,
        schema_names    =>  schemas);
    END;
    /
    
    BEGIN
      DBMS_XSTREAM_ADM.ALTER_OUTBOUND(
        server_name  => 'dbzxout',
        connect_user => 'dbzuser');
    END;
    /
    ```
    
5. Create `TEST_TABLE` table in `DBZUSER` schema and import `data/test_data.csv` into the table:
    
    ```sql
    CREATE TABLE "DBZUSER"."TEST_DATA_CSV_1" (
    	"STT" VARCHAR(32) NOT NULL PRIMARY KEY, 
    	"LINH_VUC" VARCHAR2(32), 
    	"CAP_1" VARCHAR2(64), 
    	"CAP_2" VARCHAR2(128), 
    	"CAP_3" VARCHAR2(64), 
    	"CAP_4" VARCHAR2(64), 
    	"CAP_5" VARCHAR2(1), 
    	"MAP_ITEM" VARCHAR2(64), 
    	"LINH_VUC_ORDER" NUMBER(*,0), 
    	"CAP_1_ORDER" VARCHAR2(1), 
    	"UPDATE_DATETIME" VARCHAR2(16), 
    	"TREE_ID" NUMBER(*,0), 
    	"LINH_VUC_ID" FLOAT(63), 
    	"NODE1_ID" NUMBER(*,0), 
    	"NODE2_ID" NUMBER(*,0), 
    	"NODE3_ID" NUMBER(*,0), 
    	"NODE4_ID" NUMBER(*,0), 
    	"NODE5_ID" VARCHAR2(1), 
    	"LINHVUC_SUM" VARCHAR2(16), 
    	"CAP1_SUM" VARCHAR2(32), 
    	"CAP2_SUM" VARCHAR2(16), 
    	"LINHVUC_SUM_ID" NUMBER(*,0), 
    	"CAP1_SUM_ID" NUMBER(*,0), 
    	"CAP2_SUM_ID" NUMBER(*,0), 
    	"GOIDK_ID" NUMBER(*,0)
     );
    
    # Logging must be enabled for captured tables or the database in order for data changes to capture the before state of changed database rows
    ALTER TABLE DBZUSER.TEST_DATA_CSV_1 ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
    ```
    

# Setting up Kafka

1. Spin up Kafka stack:
    
    ```bash
    docker compose -f kafka.docker-compose.yml up -d
    ```
    
2. Access Kafka UI on `[http://localhost:8080](http://localhost:8080)` and use `admin/admin` to login.
3. Navigate to the `Kafka Connect` section on the left panel and choose `Create Connector` on the top right corner, provide the name for the source connector (for example: `test_source_1`**,** in the config section type in the following and click `Create`
    
    ```
    {
    	"connector.class": "io.debezium.connector.oracle.OracleConnector",
    	"database.user": "dbzuser",
    	**"database.dbname": "ORCLCDB",
    	"tasks.max": "1",
    	"database.connection.adapter": "xstream",
    	"database.history.kafka.bootstrap.servers": "kafka-broker:9092",
    	"database.history.kafka.topic": "schema-changes.test",
    	"database.server.name": "server1",
    	"schema.include.list": "DBZUSER",
    	"database.port": "1521",
    	"database.hostname": "oracle",
    	"database.password": "dbz",
    	"database.out.server.name": "dbzxout",
    	"table.include.list": "DBZUSER.TEST_DATA_CSV_1"
    }
    ```
    
    Wait roughly 1 minutes, then click on the `Topic` section, you will see `oracle_test.DBZUSER.TEST_DATA_CSV` display in the topic list, which contain the snapshot of the table. Try to modify some records to see if any new message arrives to the topic.
    
4. Create Sink Connector to GCS `test_sink_1`:
    
    ```
    {
      "connector.class": "io.confluent.connect.gcs.GcsSinkConnector",
      "tasks.max": "1",
      "topics": "server1.DBZUSER.TEST_DATA_CSV_1",
      "gcs.bucket.name": "quannguyen_ml_bucket",
      "gcs.part.size": "5242880",
      "flush.size": "3",
      "gcs.credentials.json": "{\"type\":\"service_account\",\"project_id\":\"connect-
          1234567\",\"private_key_id\":\"omitted\",
          \"private_key\":\"-----BEGIN PRIVATE KEY-----
          \\nMIIEvAIBADANBgkqhkiG9w0BA
          \\n6MhBA9TIXB4dPiYYNOYwbfy0Lki8zGn7T6wovGS5\opzsIh
          \\nOAQ8oRolFp\rdwc2cC5wyZ2+E+bhwn
          \\nPdCTW+oZoodY\\nOGB18cCKn5mJRzpiYsb5eGv2fN\/J
          \\n...rest of key omitted...
          \\n-----END PRIVATE KEY-----\\n\",
          \"client_email\":\"pub-sub@connect-123456789.iam.gserviceaccount.com\",
          \"client_id\":\"123456789\",\"auth_uri\":\"https:\/\/accounts.google.com\/o\/oauth2\/
          auth\",\"token_uri\":\"https:\/\/oauth2.googleapis.com\/
          token\",\"auth_provider_x509_cert_url\":\"https:\/\/
          www.googleapis.com\/oauth2\/v1\/
          certs\",\"client_x509_cert_url\":\"https:\/\/www.googleapis.com\/
          robot\/v1\/metadata\/x509\/pub-sub%40connect-
          123456789.iam.gserviceaccount.com\"}",
      "storage.class": "io.confluent.connect.gcs.storage.GcsStorage",
      "format.class": "io.confluent.connect.gcs.format.avro.AvroFormat",
      "partitioner.class": "io.confluent.connect.storage.partitioner.DefaultPartitioner",
      "value.converter": "io.confluent.connect.avro.AvroConverter",
      "value.converter.schema.registry.url": "http://schema-registry:8081",
      "schema.compatibility": "NONE",
      "confluent.topic.bootstrap.servers": "kafka-broker:9092",
      "confluent.topic.replication.factor": "1"
    }
    ```
    
    Change `gcs.credentials.json` attribute to your service account credential, this service account should have Storage Admin role (using `stringify-gcp-credentials.ipynb` to formatting the file - [reference](https://docs.confluent.io/kafka-connect-gcs-sink/current/overview.html#id3))