---
title: Работа со смарт-контрактами в Tron на PHP
date: 2025-08-01
categories: [Blockchain, Tron, PHP]
tags: [tron, php, sign transaction, blockchain, tokens, trc-20] 
description: Как читать и записывать данные в смарт-контракт Tron на PHP.
---

Ранее мы уже успели немного поработать с TRC-10 смарт-контрактами, когда [активировали адрес]({% post_url 2025-07-14-tron-address-activation %}).
Продолжим дальше работать с сетью Shasta и попробуем повзаимодействовать с TRC-20 смарт-контрактом USDT ([`TG3XXyExBkPp9nzdajDZsozEu4BkaSJozs`](https://shasta.tronscan.org/#/contract/TG3XXyExBkPp9nzdajDZsozEu4BkaSJozs/code){:target="_blank"}).

```php
$fullNode = new \IEXBase\TronAPI\Provider\HttpProvider(
    'https://api.shasta.trongrid.io'
);
$solidityNode = new \IEXBase\TronAPI\Provider\HttpProvider(
    'https://api.shasta.trongrid.io'
);

$tron = new \IEXBase\TronAPI\Tron($fullNode, $solidityNode);
```

>Где взять TRC-20 в тестнете Shasta? У Tron-а есть свой [официальный discord](https://discord.com/invite/hqKvyAM){:target="_blank"}.
>Там в канале `#faucet-test-coins` можно через бота получить себе немного TRX и USDT.
>![](/assets/img/posts/tron-discord-faucet.png)
{: .prompt-tip }

## Создание объекта

Для взаимодействия со смарт-контрактом из PHP нужно создать объект контракта, передав адрес:

```php
$contract = $tron->contract('TG3XXyExBkPp9nzdajDZsozEu4BkaSJozs');  
```

>Метод `contract()` подходит только для работы с TRC-20 контрактами
{: .prompt-info }

## Чтение данных из контракта

Можно повызывать стандартные методы из интерфейса TRC-20 для получения информации о токене:

```php
echo "Name: {$contract->name()}\n"; // Name: TetherToken
echo "Symbol: {$contract->symbol()}\n"; // Symbol: USDT
```

Или всю информацию в виде массива:

```php
print_r($contract->array());
// Array
//(
//    [name] => TetherToken
//    [symbol] => USDT
//    [decimals] => 6
//    [totalSupply] => 10000000000.000000
//)
```

Получить свой баланс:

```php
$balance = $contract->balanceOf('TRcaRU8eLxUsnj2htorxZGncT9ZB4xYxQr');
echo "Balance is $balance USDT\n"; 
// Balance is 5000.000000 USDT
```

Баланс вернется сразу в виде decimal числа. Все эти вызовы были чтением контракта (данных из блокчейна) и поэтому не 
требовали создания и подписи транзакции. Теперь попробуем отправить немного токенов с одного адреса на другой.

## Отправка токенов 

Для отправки токенов с одного адреса на другой используется TRC-20 метод `transfer()`:

```php
$result = $contract->transfer('TY5vve1fL4odqFmVwB4pwCJsiy8jfGJaFe', '5');
if (!$result['result']) {
    echo 'Error: ' . $result['message'] . PHP_EOL;
} else {
    echo 'Transaction hash: ' . $result['txid'] . PHP_EOL;
}
```

>Обычно для вызова метода `transfer()` смарт-контракт USDT TRC-20 хватает `feeLimit` в 10 TRX. Но если вдруг вызов
>смарт-контракта [фейлится](https://shasta.tronscan.org/#/transaction/3073dd5a3aa82135eb9653004512a6e5509184f0603c61c570c5f6a7bb846ac1){:target="_blank"} по причине `Out of Energy`,
>то можно вручную установить этот лимит:
>```php
>$contract->setFeeLimit(20); // 20 TRX
>```
>![](/assets/img/posts/tron-out-of-energy.png)
{: .prompt-info }

Библиотека внутри сразу создает, подписывает и бродкастит транзакцию. В результате мы получаем хэш и саму подписанную транзакцию.
