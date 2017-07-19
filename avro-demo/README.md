# Debezium with Avro and Schema Registry

How to run:

```shell
# Start the topology as defined in http://debezium.io/docs/tutorial/ and with Schema Registry
docker-compose up --build

# Start MySQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register.json

# Consume messages from a topic
docker exec -it avrodemo_schema-registry_1 kafka-avro-console-consumer --zookeeper zookeeper:2181 -property schema.registry.url=http://schema-registry:8081 --from-beginning --topic dbserver1

# Shut down the cluster
docker-compose down
```
