# Confluent Platform with Datagen Source & MongoDB Sink
Create a local Confluent environment on Docker Compose, generate test data with the datagen connector and let the data sink to MongoDB.

## Instructions

Clone the repository
```
$ git clone git@github.com:senadjukic/kafka-connect-proxy-confluent-cloud.git && \
cd kafka-connect-proxy-confluent-cloud
```

Create environment variable file
```
cat > $PWD/.env <<EOF
CCLOUD_BOOTSTRAP_SERVER=pkc-xxxxx.eu-central-1.aws.confluent.cloud:9092
CCLOUD_API_KEY=
CCLOUD_API_SECRET=
CCLOUD_BOOTSTRAP_SERVER_NO_PORT=pkc-xxxxx.eu-central-1.aws.confluent.cloud
CCLOUD_TOPIC=test
CCLOUD_SCHEMA_REGISTRY=
CCLOUD_SCHEMA_REGISTRY_CREDENTIALS=user:pw
EOF
```

Run the docker-compose script
```
$ docker-compose --env-file .env up
```

Initialize MongoDB replica set
```
$ docker exec -i mongodb mongosh --eval 'rs.initiate({_id: "myuser", members:[{_id: 0, host: "mongodb:27017"}]})'
```

Create a user profile
```
$ docker exec -i mongodb mongosh << EOF
use admin
db.createUser(
{
user: "myuser",
pwd: "mypassword",
roles: ["dbOwner"]
}
)
EOF
```

Create the MongoDB sink connector:
```
$ curl -X PUT \
     -H "Content-Type: application/json" \
     -d "$(cat <<EOF
          {
            "connector.class" : "com.mongodb.kafka.connect.MongoSinkConnector",
            "key.converter": "org.apache.kafka.connect.storage.StringConverter",
            "key.converter.schema.registry.url": "$CCLOUD_SCHEMA_REGISTRY:8081",
            "key.converter.schemas.enable": "false",
            "value.converter.schemas.enable": "false",
            "value.converter": "org.apache.kafka.connect.json.JsonConverter",
            "value.converter.schema.registry.url": "$CCLOUD_SCHEMA_REGISTRY:8081",
            "tasks.max" : "1",
            "connection.uri" : "mongodb://myuser:mypassword@mongodb:27017",
            "database":"inventory",
            "collection":"customers",
            "topics":"$CCLOUD_TOPIC",
            "transforms": "dropKey",
            "transforms.dropKey.type": "io.confluent.connect.transforms.Drop\$Key",
            "transforms.dropKey.schema.behavior": "nullify"
}
EOF
)" \
     http://localhost:8083/connectors/mongodb-sink/config | jq .
```

View the records
```
$ docker exec -i mongodb mongosh << EOF
use inventory
db.customers.find().pretty();
EOF
```

Check status of the connectors
```
$ curl -s "http://localhost:8083/connectors"| \
  jq '.[]'| \
  xargs -I{connector_name} curl -s "http://localhost:8083/connectors/"{connector_name}"/status" | \
  jq -c -M '[.name,.connector.state,.tasks[].state]|join(":|:")' | \
  column -s : -t | \
  sed 's/\"//g' | \
  sort

$ curl -i -X GET http://localhost:8083/connectors/mongodb-sink/status | jq
```

Delete the connectors
```
$ curl -i -X DELETE http://localhost:8083/connectors/mongodb-sink
```