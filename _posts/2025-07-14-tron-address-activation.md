---
title: Активация Tron адреса 
date: 2025-06-24
categories: [Blockchain, Tron, PHP]
tags: [tron, php, blockchain node] 
description: Способы активации Tron адреса в PHP.
---

>Для активации адреса нужен уже активированный адрес, на котором уже есть некоторое количество TRX. Для получения
> TRX в тестнете Shasta можно использовать [faucet](https://shasta.tronex.io){:target="_blank"}.
{: .prompt-info }

## CreateAccount API

Ранее мы уже [обсуждали]({% post_url 2025-06-19-generate-tron-address-in-php %})генерацию Tron адреса и что, его нужно 
активировать. Одним из способов активации было – вызов [Create Account API](https://developers.tron.network/reference/account-createaccount){:target="_blank"}.
Попробуем это сделать. 

Итак, допустим у нас есть адрес `TRcaRU8eLxUsnj2htorxZGncT9ZB4xYxQr` с TRX на балансе и новый сгенерированный адрес `TH4STRrsWLpmFVv1jQTeB3J7ERwe7PZWsY`,  
который мы хотим активировать. Активация адреса – это транзакция, которую нужно будет создать, подписать приватным ключом и 
потом отправить в сеть. Поэтому нужно сперва указать библиотеки приватный ключ от аккаунта, с которого мы будем 
подписывать транзакции. И вызываем апи для активации адреса:

```php
$tron->setPrivateKey('YOUR-PRIVATE-KEY');
$transaction = $tron->registerAccount(
    'TRcaRU8eLxUsnj2htorxZGncT9ZB4xYxQr', 'TH4STRrsWLpmFVv1jQTeB3J7ERwe7PZWsY'
);
```

В ответ получим объект "неподписанной" транзакции в виде массива. У транзакции будет поле `txID` – идентификатор этой транзакции. 

>Далее у нас будет **1 минута**, чтобы подписать эту транзакцию и забродкастить её (отправить на ноду).
{: .prompt-warning }

```php
$signedTransaction = $tron->signTransaction($transaction);
$result = $tron->sendRawTransaction($signedTransaction);

if (!$result['result']) {
    echo 'Error: ' . $result['message'] . PHP_EOL;
} else {
    echo 'Transaction hash: ' . $result['txid'] . PHP_EOL;
}
// Transaction hash: 7a6546cda0f49753691805596b159f9eeb72895da788612bd1b99d946d9ef783
```

Метод `signTransaction()` добавляет в массив транзакции поле `signature`, в котором хранится подпись `txID` этой транзакции
нашим приватным ключом. Далее метод `sendRawTransaction()` под капотом вызывает API метод full ноды [`wallet/broadcasttransaction`](https://developers.tron.network/reference/broadcasttransaction){:target="_blank"}. 
В ответ получаем результат и хэш этой транзакции. По хэшу можно найти транзакцию на [эксплорере](https://shasta.tronscan.org/#/transaction/7a6546cda0f49753691805596b159f9eeb72895da788612bd1b99d946d9ef783){:target="_blank"}:

![](/assets/img/posts/tron-address-activation.png)

## Отправка TRX

Еще одним способом активации нового адреса является отправка TRX с уже активированного адреса. Достаточно будет отправить
0.1 TRX:

```php
$result = $tron->sendTransaction(
    'TH4STRrsWLpmFVv1jQTeB3J7ERwe7PZWsY', 0.1, 'TRcaRU8eLxUsnj2htorxZGncT9ZB4xYxQr'
);

if (!$result['result']) {
    echo 'Error: ' . $result['message'] . PHP_EOL;
} else {
    echo 'Transaction hash: ' . $result['txid'] . PHP_EOL;
}
```

Указываем куда (`TH4STRrsWLpmFVv1jQTeB3J7ERwe7PZWsY`), сколько TRX (можно указать в виде дробного числа `0.1`) и с 
какого адреса `TRcaRU8eLxUsnj2htorxZGncT9ZB4xYxQr` отправляем. Здесь под капотом сразу будет транзакция создана, 
подписана и забродкащена. В ответ мы уже получаем результат брокаста с хэшем транзакции на блокчейне. 

Если планируем все транзакции отправлять с одного адреса, то можно один раз указать свой адрес:

```php
$tron->setPrivateKey('YOUR-PRIVATE-KEY');
$tron->setAddress('TRcaRU8eLxUsnj2htorxZGncT9ZB4xYxQr');

$transaction = $tron->sendTransaction('TH4STRrsWLpmFVv1jQTeB3J7ERwe7PZWsY', 0.1);
```

## Отправка TRC-10 токена

Ещё одним вариантом активации адреса может быть отправка TRC-10 токена на этот адрес. Попробуем это сделать в 
тетснете Nile, указав в качестве ноды `https://api.nileex.io`:

```php
$fullNode = new \IEXBase\TronAPI\Provider\HttpProvider('https://api.nileex.io');
$solidityNode = new \IEXBase\TronAPI\Provider\HttpProvider('https://api.nileex.io');
$tron = new \IEXBase\TronAPI\Tron($fullNode, $solidityNode);
```

>Где взять TRC-10 токены в тестнете? Можно воспользоваться [faucet-ом для сети Nile](https://nileex.io/join/getJoinPage){:target="_blank"}.
>Там можно получить 100 `TRN` токенов.
{: .prompt-tip }
 
Имея у себя на балансе TRC-10 токены и немного TRX (для покрытия transaction fee), можно попробовать перевести немного
токенов на новый адрес:

```php
$result = $tron->sendTokenTransaction('TY5vve1fL4odqFmVwB4pwCJsiy8jfGJaFe', 10, 1005416);
if (!$result['result']) {
    echo 'Error: ' . $result['message'] . PHP_EOL;
} else {
    echo 'Transaction hash: ' . $result['txid'] . PHP_EOL;
}
```

В примере выше мы переводим 10 токенов на адрес `TY5vve1fL4odqFmVwB4pwCJsiy8jfGJaFe`. Что такое третий параметр `1005416`?
Это айди TRC-10 токена. Где его взять? Можно через API, можно прямо в [эксплорере](https://nile.tronscan.org/#/token/1005416/transfers){:target="_blank"}. 

![](/assets/img/posts/nile-trc10-asset-id.png)

>Только TRC-10 токены имеют айдишники. Для TRC-20 токенов это неактуально.
{: .prompt-info }

Вызов `sendTokenTransaction()` под капотом создаст транзакцию, подпишет и отправит на ноду. В ответ мы получим уже результат
с хэшем транзакции и данные подписанной транзакции.
