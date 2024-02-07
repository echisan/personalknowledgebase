# Paimon Testing

install hms

```bash
docker run -d -p 9083:9083 --env SERVICE_NAME=metastore \
 --name metastore-standalone  \
 apache/hive:3.1.3
```

