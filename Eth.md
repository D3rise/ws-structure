# Развертывание
1. Создание аккаунтов:  
15 раз ввести команду `geth account new --datadir data` (на одном компьютере)  
Заносить адреса в отдельный файл:  
    1. Резервный аккаунт
    2. Магазины:
        * №1
        * №2
        * №3
        * №4
        * №5
        * №6
        * №7
        * №8
        * №9
    3. Первоначальные аккаунты:
        * Банк
        * ivan
        * semen
        * petr
        * goldfish
2. Созданные ключи (`data/geth/keychain`) скопировать на вторую машину
3. Создать конфигурацию [генезис-блока](#%D0%B3%D0%B5%D0%BD%D0%B5%D0%B7%D0%B8%D1%81-%D0%B1%D0%BB%D0%BE%D0%BA) в genesis.json
4. `geth init genesis.json --datadir data`
5. Создать файл `password.txt` и написать туда пароль резервного аккаунта
6. `geth console --datadir data --http --http.api "web3,eth,personal,admin,net,miner" --http.addr "0.0.0.0" --http.corsdomain "*" --unlock "адрес резервного аккаунта" --password password.txt --nat extip:<внешний ip-адрес> --bootnodes <enode второй машины>` 

# Генезис-блок
```json
{
    "config": {
        "chainId": 15,
        "homesteadBlock": 0,
        "eip150Block": 0,
        "eip155Block": 0,
        "eip158Block": 0,
        "eip160Block": 0,
        "byzantiumBlock": 0,
        "instanbulBlock": 0,
        "constantinopleBlock": 0,
        "petersburgBlock": 0,
        "ethash": {}
    },
    "difficulty": "1",
    "gasLimit": "3000000000",
    "alloc": {
        "адрес резервного аккаунта": {
            "balance": "10000000000000000000000" // 100000 eth
        },
        ...магазины с указанным в ТЗ балансом,
        ...первоначальные аккаунты с указанным в ТЗ балансом
    }
}
```

# Контракт
_Сценарий_:
1. **Разворачиваем сеть на обеих машинах**
2. **Пишем контракт** (каждая версия наследуется от предыдующей)
3. **Пишем клиенты под новые функции контракта** (Вова - CLI, Артём - GUI)
4. **Переходим к следующей версии контракта**, повторяем шаги 2-4 до готовности решения
5. Оставляем 30 минут на **запись демонстрации**

_Пример комментариев к функциям и переменным_:
```js
/**
 * Создает пользователя
 * @param {string} username - Имя создаваемого пользователя
 * @param {string} fullName - Полное имя создаваемого пользователя
 * @param {string} password - Пароль создаваемого пользователя
*/
function createUser(string username, string fullName, string password) { ... }
```

_Глоссарий_:
1. *`payable` после декларации функции* - модификатор функции, указывающий что она может принимать какое-либо кол-во `wei` в транзакции
    * Пример отправки транзакции с `8 eth` (`8*10^18 wei`):
        ```js
        contract.methods.operateMoneyRequest("dmitrov").send({ from: "0x219aiswudjaw209haosdoalsasssdd", value: web3.utils.toWei("8", "eth") })
        ```

_Версии решения_:
1. `V1`
    * Структуры
        1. `enum Role` (перечисление ролей, индексация с нуля):
            1. `BUYER`
            2. `CASHIER`
            3. `SHOP`
            4. `PROVIDER`
            5. `BANK`
            6. `ADMIN`
        2. Структура пользователя:
            1. `string username` - имя пользователя
            2. `string fullName` - полное ФИО
            3. `string shop` - магазин, к которому принадлежит пользователь (изначально пустая строка)
            4. `bytes32 secretHash` - хеш второго пароля
            5. `Role currentRole` - текущая роль (для переключения между ролями)
            6. `Role role` - роль (фактическая, максимальная)
            7. `bool exists` - существует ли пользователь
    * Функции
        1. Аутентификация:
            1. `getUserAddress(string username)` - получение адреса для разблокировки кошелька
                * **Возвращает**:
                    1. `address <без названия>` - адрес пользователя
            2. `authenticateUser(string password)` - проверка второго пароля пользователя (отправляется с кошелька проверяемого пользователя)
                * **Возвращает**:
                    1. `bool <без названия>` - верен ли введенный пароль
        2. Регистрация:
            1. `registerUser(string username, string fullName, bytes32 secretHash)` - создание нового аккаунта (отправляется с кошелька создаваемого пользователя)
                * **Принимает**:
                    1. `address addr` - адрес кошелька владельца аккаунта (создается на стороне клиента)
                    2. `string username` - имя пользователя нового аккаунта
                    3. `string fullName` - ФИО пользователя нового аккаунта
                    4. `bytes32 secretHash` - хеш второго пароля пользователя (создается на клиенте через `web3.utils.keccak256(string)`)
        3. Личный кабинет:
            1. `getUser(address addr)` - получить информацию о пользователе
                * **Принимает**:
                    1. `address addr` - адрес искомого пользователя
                * **Возвращает**:
                    1. `string username` - имя пользователя
                    2. `string fullName` - ФИО пользователя
                    3. `string shop` - название (город) магазина, которому принадлежит пользователь (если не принадлежит, то пустая строка)
                    4. `Role role` - фактическая (максимальная) роль пользователя
                    5. `Role currentRole` - текущая роль пользователя
                    6. `uint[] reviewIds` - ID отзывов пользователя
2. `V2`
    * Структуры
        1. Структура магазина:
            1. `string city` - город, которому принадлежит магазин (название магазина)
            2. `address owner` - адрес кошелька владельца
            3. `address[] cashiers` - массив кассиров
            4. `uint[] reviewIds` - ID отзывов к магазину
            4. `bool exists` - существует ли магазин
    * Функции
        1. Получение объектов:
            1. `getShops()` - получение названий (городов) всех магазинов
                * **Возвращает**:
                    1. `string[] <без названия>` - массив названий (городов) всех магазинов
            2. `getShop(string city)` - получение информации о магазине
                * **Принимает**:
                    1. `string city` - название (город) магазина
                * **Возвращает**:
                    1. `string city` - название (город) магазина
                    2. `address owner` - адрес кошелька владельца магазина
                    3. `address[] cashiers` - адреса всех кассиров магазина
                    4. `uint[] reviewIds` - ID всех отзывов (без ответов) к магазину
                    5. `bool exists` - существует ли магазин
        2. Создание объектов:
            1. `createShop(address owner, string city)` - создание магазина
                * **Принимает**:
                    1. `address owner` - адрес кошелька владельца
                    2. `string city` - название (город) магазина
        3. Удаление объектов:
            1. `deleteShop(string city)` - удаление магазина
                * **Принимает**:
                    1. `string city` - название (город) удаляемого магазина
3. `V3`
    * Структуры
        1. Структура отзыва:
            1. `uint id` - ID отзыва
            2. `uint rate` - оценка магазина (если это ответ - то является 0)
            3. `uint[] children` - ID дочерних отзывов (ответов)
            4. `address author` - адрес автора
            5. `address[] likes` - адреса пользователей, оставивших лайк
            6. `address[] dislikes` - адреса пользователей, оставивших дизлайк
            7. `string shop` - название магазина (город)
            8. `string content` - содержание отзыва (если есть, иначе пустая строка)
            9. `bool exists` - существует ли отзыв
    * Функции
        1. `getReview(uint id)` - получение отзыва
            * **Принимает**:
                1. `uint id` - ID запрашиваемого отзыва
            * **Возвращает**:
                1. `address author` - адрес автора отзыва
                2. `uint id` - ID отзыва
                3. `uint rate` - оценка оставленная магазина (если ответ - то 0)
                4. `uint likes` - количество лайков
                5. `uint dislikes` - количество дизлайков
                6. `uint[] children` - дочерние отзывы (ответы)
                7. `string shop` - название (город) магазина
                8. `string content` - содержание отзыва (если отсутствует - то пустая строка)
                9. `bool exists` - существует ли отзыв
        2. `createReview(string shop, string content, uint rate, uint parentId)` - создание отзыва или ответа
            * **Разрешенные роли**:
                1. **Покупатель `(0 BUYER)`** - *может оставлять как оценки, так и ответы*
                2. **Продавец `(1 CASHIER)`** - *может оставлять только ответы на оценки в __своем__ магазине*
            * **Принимает**:
                1. `string shop` - название (город) магазина, к которому создается отзыв
                2. `string content` - содержание отзыва (если отсутствует - пустая строка)
                3. `uint rate` - оставляемая оценка (от `1` до `10`, если ответ - то `0`)
                4. `uint parentId` - ID родительского отзыва (если отсутствует - то `0`)
4. `V4`
5. `V5`
    * Структуры
        1. Структура запроса на изменение роли (повышение и понижение):
            1. `address sender` - адрес отправившего запрос
            2. `Role role` - требуемая роль
            3. `string shop` - требуемый магазин (если роль не требует магазина, то пустая строка)
            4. `bool exists` - существует ли запрос
    * Функции
        1. Получение объектов
            1. `getElevateRequests()` - получение адресов пользователей, оставивших запрос на изменение роли
                * **Разрешенные роли**:
                    1. Администратор `(5 ADMIN)`
                * **Возвращает**:
                    1. `address[] <без названия>` - адреса пользователей, которые оставили запрос на изменение роли
            2. `getElevateRequest(address sender)` - получить информацию о запросе на повышение
                * **Разрешенные роли**:
                    1. Администратор `(5 ADMIN)`
                * **Принимает**:
                    1. `address sender` - адрес пользователя, которому принадлежит запрос
                * **Возвращает**:
                    1. `address author` - адрес пользователя, которому принадлежит запрос
                    2. `Role role` - запрашиваемая роль
                    3. `string shop` - запрашиваемый магазин (если роль не предполагает наличие магазина - то пустая строка)
                    4. `bool exists` - существует ли запрос
            3. ``
6. `V6`
    * Структуры:
        1. Структура запроса на займ:
            1. `address owner` - адрес владельца магазина
            2. `string shop` - название (город) магазина
            3. `uint value` - количество запрашиваемых средств в `wei`
            4. `bool exists` - существует ли запрос
    * Функции:
        1. Получение объектов:
            1. `getMoneyRequests()` - получение названий (городов) всех магазинов, которые отправили запрос на займ в банк
                * **Разрешенные роли**:
                    1. Банк `(4 BANK)`
                * **Возвращает**:
                    1. `string[] <без названия>` - массив названий (городов) всех магазинов  которые отправили запрос на займ
            2. `getMoneyRequest(string shop)` - получение информации о займе
                * **Разрешенные роли**:
                    1. Банк `(4 BANK)`
                * **Принимает**:
                    1. `string shop` - название (город) магазина, который отправил запрос
                * **Возвращает**:
                    1. `string city` - название (город) магазина, который отправил запрос
                    2. `address owner` - адрес владельца магазина
                    3. `uint value` - количество запрашиваемых средств (в `wei`)
                    4. `bool exists` - существует ли запрос
        2. Создание объектов:
            1. `newMoneyRequest(uint value)`
                * **Разрешенные роли**:
                    1. Магазин `(3 SHOP)`
                * **Принимает**:
                    1. `uint value` - количество запрашиваемых средств в `wei`
        3. Изменение объектов:
            1. `operateMoneyRequest(string shop, bool approve) payable` - изменить состояние запроса на займ
                * **Разрешенные роли**:
                    1. Магазин `(3 SHOP)`
                * **Принимает**:
                    1. `string shop` - название магазина, отправившего запрос
                    2. `bool approve` - `true` если принимаем запрос, `false` - если отклоняем.
                        * Если принимаем - нужно отправить вместе с вызовом функции то кол-во `wei`, которое указано в запросе на займ, иначе - не отправляем