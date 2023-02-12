# Анализ на предмет CAP

## DragonFly

В основной [документации](https://github.com/dragonflydb/dragonfly/blob/main/README.md) репликация заявлена, как фича в роадмапе.
Однако, в релизе 0.14.0 промелькнул такой релизноут:
> Replication is almost ready - one can set up a master/slave setup with a usual "replicaof" command on a secondary replica (FAILOVER is not supported yet)

Для проверки сценария использован docker-compose. Используются 3 контейнера: два инстанса Dragonfly (df1 и df2) и один инстанс redis клиента, для подключения к :
1. Поднимем контейнеров `docker-compose -f dragonfly.yml up -d`.
2. Подключимся к инстансу df1 `docker exec -it redis redis-cli -h df1`. Эту сессию назовем сессией 1.
3. Подключимся к инстансу df2 `docker exec -it redis redis-cli -h df2`. Эту сессию назовем сессией 2.
4. В сессии 2 сделаем df2 репликой df1 `replicaof df1 6379`. Получим `OK`.
5. В сессии 1 создадим пару ключ/значение `set hello world`.
6. В сессии 2 получим значение по ключу `get hello`. Получим `"world"`. Репликация работает.
7. Отсоединим df2 от сети df1-df2 `docker network disconnect df1-df2 df2`.
8. В сессии 1 изменим значение ключа hello `set hello notworld`.
9. В сессии 2 получим значение ключа hello `get hello`. Получим старое значение `world`.
10. Вернем df2 в сеть df1-df2 `docker network connect df1-df2 df2`.
11. Подождем минуту для синхронизации реплики.
11. В сессии 2 получим значение ключа hello `get hello`. Получим новое значение `notworld`.

Итак, в условиях разрыва соединения система продолжает работать в штатном режиме, но данные из реплик могут быть не самыми свежими. Таким образом, эта база данных относится к **AP**.