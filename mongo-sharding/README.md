## Инструкция по инициализации шардированного кластера MongoDB в Docker

Данная инструкция описывает процесс инициализации шардированного кластера MongoDB, развернутого в Docker, а также наполнение его тестовыми данными и проверку распределения этих данных по шардам.

**Предполагается, что у вас запущены контейнеры MongoDB, как описано в исходном вопросе: `configSrv`, `shard1`, `shard2`, и `mongos_router`.**

### Шаг 1: Инициализация сервера конфигурации (`configSrv`)

1.  Подключитесь к контейнеру `configSrv` с помощью `mongosh`:

    ```bash
    docker exec -it configSrv mongosh --port 27017
    ```

2.  Инициализируйте сервер конфигурации как реплика сет:

    ```javascript
    rs.initiate(
      {
        _id : "config_server",
           configsvr: true,
        members: [
          { _id : 0, host : "configSrv:27017" }
        ]
      }
    );
    ```

3.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

### Шаг 2: Инициализация шардов (`shard1` и `shard2`)

1.  Подключитесь к контейнеру `shard1` с помощью `mongosh`:

    ```bash
    docker exec -it shard1 mongosh --port 27018
    ```

2.  Инициализируйте шард как реплика сет:

    ```javascript
    rs.initiate(
        {
          _id : "shard1",
          members: [
            { _id : 0, host : "shard1:27018" }
          ]
        }
    );
    ```

3.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

4.  Подключитесь к контейнеру `shard2` с помощью `mongosh`:

    ```bash
    docker exec -it shard2 mongosh --port 27019
    ```

5.  Инициализируйте шард как реплика сет:

    ```javascript
    rs.initiate(
        {
          _id : "shard2",
          members: [
            { _id : 0, host : "shard2:27019" }
          ]
        }
      );
    ```

6.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

### Шаг 3: Инициализация роутера (`mongos_router`) и добавление шардов

1.  Подключитесь к контейнеру `mongos_router` с помощью `mongosh`:

    ```bash
    docker exec -it mongos_router mongosh --port 27020
    ```

2.  Добавьте шарды в кластер через роутер:

    ```javascript
    sh.addShard( "shard1/shard1:27018");
    sh.addShard( "shard2/shard2:27019");
    ```

3.  Включите шардирование для базы данных `somedb`:

    ```javascript
    sh.enableSharding("somedb");
    ```

4.  Задайте ключ шардирования для коллекции `somedb.helloDoc`.  В данном случае используется хешированное шардирование по полю `name`:

    ```javascript
    sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
    ```

### Шаг 4: Заполнение тестовыми данными

1.  Используйте базу данных `somedb`:

    ```javascript
    use somedb
    ```

2.  Вставьте 1000 документов в коллекцию `helloDoc`:

    ```javascript
    for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
    ```

3.  Подтвердите, что в коллекции 1000 документов:

    ```javascript
    db.helloDoc.countDocuments()
    ```

4.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

### Шаг 5: Проверка распределения данных по шардам

1.  Подключитесь к контейнеру `shard1` с помощью `mongosh`:

    ```bash
     docker exec -it shard1 mongosh --port 27018
    ```

2.  Используйте базу данных `somedb`:

    ```javascript
    use somedb;
    ```

3.  Подсчитайте количество документов в коллекции `helloDoc`:

    ```javascript
    db.helloDoc.countDocuments();
    ```

    Ожидаемый результат: приблизительно 492 документа.

4.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

5.  Подключитесь к контейнеру `shard2` с помощью `mongosh`:

    ```bash
    docker exec -it shard2 mongosh --port 27019
    ```

6.  Используйте базу данных `somedb`:

    ```javascript
    use somedb;
    ```

7.  Подсчитайте количество документов в коллекции `helloDoc`:

    ```javascript
    db.helloDoc.countDocuments();
    ```

    Ожидаемый результат: приблизительно 508 документов.

8.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

**Заключение:**

После выполнения этих шагов, вы должны увидеть, что данные были распределены между шард1 и шард2.  Точное количество документов на каждом шарде может незначительно отличаться в зависимости от используемого ключа шардирования и его распределения.