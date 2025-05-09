## Инструкция по инициализации шардированного кластера MongoDB в Docker

Данная инструкция описывает процесс инициализации шардированного кластера с 2мя репликами MongoDB, развернутого в Docker, а также наполнение его тестовыми данными и проверку распределения этих данных по шардам.


### Шаг 1: Инициализация сервера конфигурации (`configSrv1`)

1.  Подключитесь к контейнеру `configSrv1` с помощью `mongosh`:

    ```bash
    docker exec -it configSrv1 mongosh --port 27017
    ```

2.  Инициализируйте сервер конфигурации как реплика сет:

    ```javascript
    rs.initiate(
      {
        _id : "config_server",
           configsvr: true,
        members: [
          { _id : 0, host : "configSrv1:27017" }
        ]
      }
    );
    ```

3.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

### Шаг 2: Инициализация шардов (`shard1-a` и `shard2-a`)

1.  Подключитесь к контейнеру `shard1-a` с помощью `mongosh`:

    ```bash
    docker exec -it shard1-a mongosh
    ```

2.  Инициализируйте шард как реплика сет c 2мя репликами:

    ```javascript
    rs.initiate(
        {
          _id : "shard1",
          members: [
                { _id: 0, host: "shard1-a:27017" },
                { _id: 1, host: "shard1-b:27017" },
                { _id: 2, host: "shard1-c:27017" },
          ]
        }
    );
    ```

3.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

4.  Подключитесь к контейнеру `shard2-a` с помощью `mongosh`:

    ```bash
    docker exec -it shard2-a mongosh
    ```

5.  Инициализируйте шард как реплика сет c 2мя репликами:

    ```javascript
    rs.initiate(
        {
          _id : "shard2",
          members: [
                { _id: 0, host: "shard2-a:27017" },
                { _id: 1, host: "shard2-b:27017" },
                { _id: 2, host: "shard2-c:27017" },
          ]
        }
      );
    ```

6.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

### Шаг 3: Инициализация роутера (`mongos1`) и добавление шардов

1.  Подключитесь к контейнеру `mongos1` с помощью `mongosh`:

    ```bash
    docker exec -it mongos1 mongosh
    ```

2.  Добавьте шарды в кластер через роутер:

    ```javascript
    sh.addShard("shard1/shard1-a:27017");
    sh.addShard("shard2/shard2-a:27017");
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

### Шаг 5: Проверка распределения данных по шардам и репликам

1.  Подключитесь к контейнеру `shard1-a` с помощью `mongosh`:

    ```bash
     docker exec -it shard1-a mongosh
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

5.  Подключитесь к контейнеру `shard1-b` с помощью `mongosh`:

    ```bash
     docker exec -it shard1-a mongosh
    ```

6.  Используйте базу данных `somedb`:

    ```javascript
    use somedb;
    ```

7.  Подсчитайте количество документов в коллекции `helloDoc`:

    ```javascript
    db.helloDoc.countDocuments();
    ```

    Ожидаемый результат: приблизительно 492 документов.

8.  Выйдите из `mongosh`:

    ```javascript
    exit();
    ```

**Заключение:**

После выполнения этих шагов, вы должны увидеть, что данные были распределены между шард1 и шард2.  Точное количество документов на каждом шарде может незначительно отличаться в зависимости от используемого ключа шардирования и его распределения.