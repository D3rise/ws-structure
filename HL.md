## Инициализация организаций:

1. Поднятие контейнеров центров сертификации:
   1. `docker-compose up ca-org0 ca-org1 ca-org2 ca-org3`
1. Создание папки для хранения всех данных всех организаций (в нашем случае `orgs`)
1. Цикл по ВСЕМ организациям (`i = 0..3`)
   1. Создание папок MSP:
      1. `mkdir orgs/org$i orgs/org$i/msp`
      2. `mkdir orgs/org$i/msp/{admincerts,cacerts,users}`
   2. Выпуск сертификатов администраторов организаций:
      1. `fabric-ca-client enroll -u http://admin:adminpw@<адрес центра сертификации> -H orgs/org$i/admin`
   3. Копирование необходимых сертификатов в нужные папки MSP и admincerts; создание папки users:
      1. `cp orgs/ca/org$i/ca-cert.pem orgs/org$i/msp/cacerts`
      2. `cp orgs/org$i/admin/msp/signcerts/cert.pem orgs/org$i/admincerts`
      3. `mkdir orgs/org$i/admin/msp/admincerts`
      4. `cp orgs/org$i/admin/msp/signcerts/cert.pem orgs/org$i/admin/msp/admincerts`
      5. `mkdir orgs/org$i/admin/msp/users`

## Создание пиров:

1. Регистрация orderer:
   1. `fabric-ca-client register -u http://<адрес центра сертификации ordering организации> -H orgs/org0/admin --id.type orderer --id.name orderer --id.secret ordererpw`
2. Выпуск сертификата orderer:
   1. `fabric-ca-client enroll -u http://orderer:ordererpw@<адрес центра сертификации ordering организации> -H orgs/org0/orderer`
3. Создание admincerts и копирование сертификата администратора:
   1. `mkdir orgs/org0/orderer/msp/admincerts`
   2. `cp orgs/org0/admin/msp/signcerts/cert.pem`
4. Цикл по организациям-участникам сети (все, кроме ordering: `i = 1..3`)
   1. регистрация сертификатов пиров:
      1. `fabric-ca-client register -u http://<адрес центра сертификации организации> -H orgs/org$i/admin --id.name peer --id.type peer --id.secret peerpw`
   2. выпуск сертификатов пиров:
      1. `fabric-ca-client enroll -u http://peer:peerpw@<адрес центра сертификации организации> -H orgs/org$i/peer`
   3. создание admincerts и копирование сертификата администратора организации
      1. `mkdir orgs/org$i/peer/msp/admincerts`
      2. `cp orgs/org$i/admin/msp/signcerts/cert.pem orgs/org$i/peer/msp/admincerts`
5. Создание общей папки для всех организаций:
   1. `mkdir orgs/common`
6. Создание генезис-блока и блока создания канала для организаций-участников:
   1. `configtxgen -profile WSRGenesis -channelID syschannel -outputBlock orgs/org$i/orderer/genesis.block`
   2. `configtxgen -profile WSR -channelID wsr -outputCreateChannelTx orgs/common/wsr.tx`
7. Создание блоков с информацией об Anchor-пирах:
   1. `configtxgen -profile WSR -channelID wsr -outputAnchorPeersUpdate orgs/common/org1anchors.tx -asOrg Bank`
   2. `configtxgen -profile WSR -channelID wsr -outputAnchorPeersUpdate orgs/common/org2anchors.tx -asOrg Shops`
   3. `configtxgen -profile WSR -channelID wsr -outputAnchorPeersUpdate orgs/common/org3anchors.tx -asOrg Users`
8. Создание блока с транзакцией создания канала для организаций-участников:
   1. `docker exec -ti cli-org1 peer channel create -f wsr.tx -c wsr --outputBlock wsr.block -o orderer:7050`
9. Цикл по организациям-участникам (`i = 1..3`)
   1. Присоединение к каналу: `docker exec -ti cli-org$i peer channel join -b wsr.block -o orderer:7050`
   2. Создание транзакции с информацией об Anchor-пирах: `docker exec -ti cli-org$i peer channel update -f orgs${i}anchors.tx -o orderer:7050 -c wsr`
