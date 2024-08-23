# mongo-sharding-cache

Запуск MongoDB в режиме шардинга: 1 роутер, 1 сервер конфигурации, 2 шарда по 3 реплики, 6 кешей.

## Как запустить

Запускаем mongodb и приложение

docker compose up -d

# Подключитесь к серверу конфигурации и сделайте инициализацию:

docker exec -it configSrv mongosh --port 27027

rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27027" }
    ]
  }
)

exit

Инициализируйте шарды:
# Подключаемся к любому контейнеру:
docker exec -it shard1 mongosh --port 27018

Создаём набор реплик в командной оболочке mongosh:
rs.initiate({_id: "shard1", members: [
{_id: 0, host: "shard1:27018"},
{_id: 1, host: "shard1-1:27024"},
{_id: 2, host: "shard1-2:27025"},
{_id: 3, host: "shard1-3:27026"}
]}) 

exit

# Подключаемся к любому контейнеру:
docker exec -it shard2 mongosh --port 27019

Создаём набор реплик в командной оболочке mongosh:
rs.initiate({_id: "shard2", members: [
{_id: 0, host: "shard2:27019"},
{_id: 1, host: "shard2-1:27021"},
{_id: 2, host: "shard2-2:27022"},
{_id: 3, host: "shard2-3:27023"}
]}) 

exit

# Инцициализируйте роутер:
docker exec -it mongos_router mongosh --port 27020

sh.addShard( "shard1/shard1:27018")
sh.addShard( "shard2/shard2:27019")

sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )

exit

# Подключаемся к любой ноде и выполняем команду для создания кластера:
docker exec -it redis_1 bash
echo "yes" | "redis-cli" --cluster create   173.17.0.2:6379   173.17.0.3:6379   173.17.0.4:6379   173.17.0.5:6379   173.17.0.6:6379   173.17.0.27:6379  --cluster-replicas 1

Также можно выполнить команду для получения информации о кластере:
redis-cli cluster nodes
exit

# роутер наполняем тестовыми данными:

docker exec -it mongos_router mongosh --port 27020

use somedb
for(var i = 0; i < 2000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})

db.helloDoc.countDocuments() 
exit


## Как проверить
Сделайте проверку на первом шарде:
docker exec -it shard1 mongosh --port 27018
use somedb
db.helloDoc.countDocuments()
exit

Сделайте проверку на втором шарде:
docker exec -it shard2 mongosh --port 27019
use somedb
db.helloDoc.countDocuments()
exit

docker exec -it mongos_router mongosh --port 27020
use somedb
db.helloDoc.countDocuments()
exit

docker exec -it shard1-1 mongosh --port 27024
use somedb
db.helloDoc.countDocuments()
exit
