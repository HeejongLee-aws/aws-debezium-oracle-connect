# Oracle 19c Database Debezium 이용한 데이터 마이그래이션(CDC)


## Oracle 19c install
- 아래 비디오를 참고하시면 AWS EC2 환경에서 Oracld 19c 를 설치 할 수 있습니다.
- https://www.youtube.com/watch?v=GENWOMuQAis

### Create Logminer User
```
sqlplus sys/<password>@//localhost:1521/<SID> as sysdba
  CREATE TABLESPACE logminer_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/logminer_tbs.dbf'
    SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
  exit;

GRANT CREATE SESSION TO c##dbzuser CONTAINER=ALL;
GRANT SET CONTAINER TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$DATABASE to c##dbzuser CONTAINER=ALL;
GRANT FLASHBACK ANY TABLE TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ANY TABLE TO c##dbzuser CONTAINER=ALL;
GRANT SELECT_CATALOG_ROLE TO c##dbzuser CONTAINER=ALL;
GRANT EXECUTE_CATALOG_ROLE TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ANY TRANSACTION TO c##dbzuser CONTAINER=ALL;
GRANT LOGMINING TO c##dbzuser CONTAINER=ALL;

GRANT CREATE TABLE TO c##dbzuser CONTAINER=ALL;
GRANT LOCK ANY TABLE TO c##dbzuser CONTAINER=ALL;
GRANT CREATE SEQUENCE TO c##dbzuser CONTAINER=ALL;

GRANT EXECUTE ON DBMS_LOGMNR TO c##dbzuser CONTAINER=ALL;
GRANT EXECUTE ON DBMS_LOGMNR_D TO c##dbzuser CONTAINER=ALL;

GRANT SELECT ON V_$LOG TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOG_HISTORY TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOGMNR_LOGS TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOGMNR_CONTENTS TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOGMNR_PARAMETERS TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOGFILE TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$ARCHIVED_LOG TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO c##dbzuser CONTAINER=ALL;

exit;
```

## Configuration Database
```
ALTER TABLE awsuser.employee ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
```


## Dockefile 작성
- Debezium 의 Connector Docker 이미지의 경우, Oracle 19의 ojdbc8.jar 와 xstream.jar 를 포함하고 있지 않습니다.
- Oracle Connector 로 사용하기 위해서는 아래와 같이 Dockerfile 을 작성하고, Docker Image 를 Build 해야합니다.
```
ARG DEBEZIUM_VERSION
FROM debezium/connect:$DEBEZIUM_VERSION
ENV KAFKA_CONNECT_JDBC_DIR=$KAFKA_CONNECT_PLUGINS_DIR/kafka-connect-jdbc
ENV INSTANT_CLIENT_DIR=/instant_client/

USER root
RUN yum -y install libaio && yum clean all

USER kafka
# Deploy Oracle client and drivers

#COPY C:\instantclient_19_12\* $INSTANT_CLIENT_DIR
COPY C:\instantclient_19_12\xstreams.jar /kafka/libs
COPY C:\instantclient_19_12\ojdbc8.jar /kafka/libs
```


## Docker build
```docker build --tag debezium/oracle19c-connect:1.6 . --build-arg DEBEZIUM_VERSION=1.6```
- Debezium 버전을 파라매터로 넘겨준다.

## Test

### String Zookeeper
```docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:1.6```

### Starting Kafka
```docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:1.6```

### Starting Debezium Connector for Oracle
```docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka debezium/oracle19c-connect:1.6```

### view change events
```docker run -it --rm --name watcher --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:1.6 watch-topic -a -k server1.AWSUSER.EMPLOYEE```
