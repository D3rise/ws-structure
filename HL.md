## Инициализация организаций:

1. Поднятие контейнеров центров сертификации:
   - `docker-compose up ca-org0 ca-org1 ca-org2 ca-org3`
1. Создание папки для хранения всех данных всех организаций (в нашем случае `orgs`)
1. Цикл по ВСЕМ организациям (`i = 0..3`)
   1. Создание папок MSP:
      - `mkdir orgs/org$i orgs/org$i/msp`
      - `mkdir orgs/org$i/msp/{admincerts,cacerts,users}`
   2. Выпуск сертификатов администраторов организаций:
      - `fabric-ca-client enroll -u http://admin:adminpw@<адрес центра сертификации> -H orgs/org$i/admin`
   3. Копирование необходимых сертификатов в нужные папки MSP и admincerts; создание папки users:
      - `cp orgs/ca/org$i/ca-cert.pem orgs/org$i/msp/cacerts`
      - `cp orgs/org$i/admin/msp/signcerts/cert.pem orgs/org$i/admincerts`
      - `mkdir orgs/org$i/admin/msp/admincerts`
      - `cp orgs/org$i/admin/msp/signcerts/cert.pem orgs/org$i/admin/msp/admincerts`
      - `mkdir orgs/org$i/admin/msp/users`

## Создание пиров:

1. Регистрация orderer:
   - `fabric-ca-client register -u http://<адрес центра сертификации ordering организации> -H orgs/org0/admin --id.type orderer --id.name orderer --id.secret ordererpw`
2. Выпуск сертификата orderer:
   - `fabric-ca-client enroll -u http://orderer:ordererpw@<адрес центра сертификации ordering организации> -H orgs/org0/orderer`
3. Создание admincerts и копирование сертификата администратора:
   - `mkdir orgs/org0/orderer/msp/admincerts`
   - `cp orgs/org0/admin/msp/signcerts/cert.pem`
4. Цикл по организациям-участникам сети (все, кроме ordering: `i = 1..3`)
   - регистрация сертификатов пиров:
     - `fabric-ca-client register -u http://<адрес центра сертификации организации> -H orgs/org$i/admin --id.name peer --id.type peer --id.secret peerpw`
   - выпуск сертификатов пиров:
     - `fabric-ca-client enroll -u http://peer:peerpw@<адрес центра сертификации организации> -H orgs/org$i/peer`
   - создание admincerts и копирование сертификата администратора организации
     - `mkdir orgs/org$i/peer/msp/admincerts`
     - `cp orgs/org$i/admin/msp/signcerts/cert.pem orgs/org$i/peer/msp/admincerts`
5. Создание общей папки для всех организаций:
   - `mkdir orgs/common`
6. Создание генезис-блока и блока создания канала для организаций-участников:
   - `configtxgen -profile WSRGenesis -channelID syschannel -outputBlock orgs/org$i/orderer/genesis.block`
   - `configtxgen -profile WSR -channelID wsr -outputCreateChannelTx orgs/common/wsr.tx`
7. Создание блоков с информацией об Anchor-пирах:
   - `configtxgen -profile WSR -channelID wsr -outputAnchorPeersUpdate orgs/common/org1anchors.tx -asOrg Bank`
   - `configtxgen -profile WSR -channelID wsr -outputAnchorPeersUpdate orgs/common/org2anchors.tx -asOrg Shops`
   - `configtxgen -profile WSR -channelID wsr -outputAnchorPeersUpdate orgs/common/org3anchors.tx -asOrg Users`
8. Создание блока с транзакцией создания канала для организаций-участников:
   - `docker exec -ti cli-org1 peer channel create -f wsr.tx -c wsr --outputBlock wsr.block -o orderer:7050`
9. Цикл по организациям-участникам (`i = 1..3`)
   - Присоединение к каналу: `docker exec -ti cli-org$i peer channel join -b wsr.block -o orderer:7050`
   - Создание транзакции с информацией об Anchor-пирах: `docker exec -ti cli-org$i peer channel update -f orgs${i}anchors.tx -o orderer:7050 -c wsr`
