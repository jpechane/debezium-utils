# Debezium with Confluent images

How to run:

```shell
# Start the topology as defined in http://debezium.io/docs/tutorial/
docker-compose up --build

# Start MySQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register.json

# Consume messages from a topic
docker exec -it confluentdemo_kafka_1 kafka-console-consumer --bootstrap-server kafka:9092 --from-beginning --topic dbserver1

# Shut down the cluster
docker-compose down
```
