---
title: Взаимодействие с Tron нодой из PHP 
date: 2025-07-01
categories: [Blockchain, Tron, PHP]
tags: [tron, php, blockchain node]
description: Разбираем примеры работы с Tron нодой из PHP, типы нод и получение баланса адреса.
---

### Библиотека

Для работы с блокчейном Tron будем использовать библиотеку [iexbase/tron-api](https://github.com/iexbase/tron-api){:target="_blank"}:

```bash
composer require iexbase/tron-api --ignore-platform-reqs
```

>Да, библиотека довольно старая (последний релиз в 2022 году). Но в принципе в ней всё что нужно есть и она поддерживает
>PHP 8. По сути библиотека скорее мертва, чем жива: последний pull request был вмержен в 2021 году. Есть альтернатива в 
>виде [Fenguoz/tron-php](https://github.com/Fenguoz/tron-php){:target="_blank"}. Она более свежая: последний релиз был в
>2024 году, но она например не умеет отправлять TRC10 токены и не умеет в смарт-контракты. 
>
>Можно вообще использовать любой HTTP-клиент (например Guzzle) и просто делать API-запросы к ноде.
{: .prompt-info }


### Типы нод

У блокчейна Tron есть три типа нод: 
- **Super nodes** генерируют новые блоки, а так же валидируют и записывают транзакции в эти блоки. Информация с таких нод общедоступна для всех пользователей через эксплореры, например Tronscan.
- **Full nodes** осуществляют полный контроль над on-chain данными (хранят их и синхронизируют), включая real-time обновления, бродакст транзакций и предоставление API.
- **Solidity nodes** интегрируют данные из соответствующих full нод и предоставляют API. Синхронизируют только solidified блоки.

По умолчанию библиотека в качестве нод использует `https://api.trongrid.io`, поэтому если мы планируем работать с 
`trongrid`, то получить номер последнего блока можно следующим образом:

```php
$tron = new \IEXBase\TronAPI\Tron();
$blockData = $tron->getCurrentBlock();
echo "Last block number: " . $blockData['block_header']['raw_data']['number'];
```

Если планируем использовать свои ноды, то нужно указать это явно. Например, если хотим работать с тестнетом Shasta:
```php
$shastaFullNode = new \IEXBase\TronAPI\Provider\HttpProvider(
    'https://api.shasta.trongrid.io'
);
$shastaSolidityNode = new \IEXBase\TronAPI\Provider\HttpProvider(
    'https://api.shasta.trongrid.io'
);

$tronShasta = new \IEXBase\TronAPI\Tron($shastaFullNode, $shastaSolidityNode);
```

>Под капотом вызывается jsonrpc метод solidity ноды [`walletsolidity/getnowblock`](https://developers.tron.network/reference/getnowblock){:target="_blank"}.
>По сути объект `Tron` - просто обертка с хэлперами методами для доступа сразу ко всем нодам. Например, запрос для получения
>блока `getCurrentBlock()` можно переписать с запросом к ноде:
>```php
>$blockData = $solidityNode->request('walletsolidity/getnowblock');
>$blockNumber = $blockData['block_header']['raw_data']['number'];
>```
{: .prompt-info }

### Получение баланса

В блокчейне Tron можно получить TRX баланс адреса через апи метод solidity ноды [`walletsolidity/getaccount`](https://developers.tron.network/reference/walletsolidity-getaccount){:target="_blank"}.
В нашем случае библиотека предоставляет удобный хэлпер для этого:

```php
$balance = $tron->getBalance('TXHy2ftx3pxbvba3LRm8NoDLkeg949L7Ze');
echo $balance . PHP_EOL; // 153492332146
```

По умолчанию баланс вернется в минимальных единицах. Например, если баланс адреса `153,492.332146 TRX`, то мы получим
число `153492332146`. Чтоб получить красиво отформатированное число (в виде decimal), нужно вторым аргументом передать `true`:

```php
$balance = $tron->getBalance('TXHy2ftx3pxbvba3LRm8NoDLkeg949L7Ze', true);
echo $balance . PHP_EOL; // 153,492.332146
```

### Информация о ноде

Ради интереса модно получить данные о самой ноде, с которой мы общаемся. Хэлпера для этого нет, поэтому нужно будет
сделать запрос напрямую. Будем вызывать API [`walletsolidity/getnodeinfo`](https://developers.tron.network/reference/getnodeinfo-1){:target="_blank"}:

```php
$nodeInfo = $solidityNode->request('walletsolidity/getnodeinfo');
echo $nodeInfo['machineInfo']['osName']; // "Linux 3.10.0-1160.49.1.el7.x86_64"
```

Код выше выведет OS, которая работает на solidity ноде.






