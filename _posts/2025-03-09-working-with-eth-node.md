---
title: Взаимодействие с Ethereum нодой из PHP
date: 2025-03-09
categories: [Blockchain, Ethereum, PHP]
tags: [ethereum, php, blockchain node] 
---

По сути вся работа с блокчейном в PHP - это так или иначе взаимодействие с блокчейн нодой. Все подписанные транзакции
отправляются (бродкастятся) на неё и все данные из блокчейна также можно получить у ноды. Обычно у ноды есть свой json-rpc интерфейс,
через который с ней можно взаимодействовать. 

>[Документация](https://ethereum.org/en/developers/docs/apis/json-rpc/){:target="_blank"} по JSON-RPC API Ethereum ноды.
{: .prompt-info }

### Библиотека

Так как общение с нодой - это просто HTTP-запросы, то у нас есть два варианта. Можно либо использовать готовую библиотеку,
которая формирует нужный payload и парсит ответы. Либо использовать любой HTTP-клиент и делать всё вручную. Рассмотрим вариант с
уже готовой библиотекой [Simple-Web3-Php](https://github.com/drlecks/Simple-Web3-Php){:target="_blank"}:

```bash
composer require drlecks/simple-web3-php
```

И попробуем подключиться к публичной ноде `https://eth.public-rpc.com` и узнать номер текущего блока в блокчейне:

```php
use SWeb3\SWeb3;

$node = new SWeb3('https://eth.public-rpc.com');
$res = $node->call('eth_blockNumber', []);
```

Метод `call(string $method, $params = null)` позволяет вызывать json-rpc метод у ноды. В нашем случае вызываем 
[eth_blocknumber](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_blocknumber){:target="_blank"},
у которого нет параметров. В ответ получим объект stdClass с телом json-rpc ответа:

```
stdClass Object
(
    [id] => 1
    [jsonrpc] => 2.0
    [result] => 0x14fcf95
)
```

Нам интересно поле `result`, но важно помнить, что данные нода вернет в 16-ом формате. Поэтому, чтобы получить номер
блока в читаемом виде - нужно будет явно его привести в 10-ую систему:

```php
use SWeb3\SWeb3;
use SWeb3\Utils;

$node = new SWeb3('https://eth.public-rpc.com');
$res = $node->call('eth_blockNumber', []);
$block = Utils::hexToBn($res->result);
echo 'Block number: ' . $block->toString() . PHP_EOL;

// Block number: 22008077
```

### Получение баланса

Теперь попробуем получить баланс ([`eth_getBalance()`](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_getbalance){:target="_blank"}) 
какого-нибудь адреса ([0x4838B106FCe9647Bdf1E7877BF73cE8B0BAD5f97](https://etherscan.io/address/0x4838b106fce9647bdf1e7877bf73ce8b0bad5f97){:target="_blank"}:

```php
$res = $node->call('eth_getBalance', ['0x4838B106FCe9647Bdf1E7877BF73cE8B0BAD5f97', 'latest']);
$balance = Utils::hexToBn($res->result);
echo 'Balance: ' . $balance->toString() . PHP_EOL;

// Balance: 16857795621056046838
```

В результате получаем число, которое отличается от, того что мы видим на эксплорере в `ETH BALANCE`:

![](/assets/img/posts/eth-address-balance-explorer.png)

Почему так происходит? Дело в том, что нода возвращает балансы в wei (наименьший номинал ETH).

>1 ETH = 1,000,000,000,000,000,000 WEI.
{: .prompt-info }  

Чтобы из Wei получить ETH нужно явно привести полученное число wei в eth:

```php
$res = $node->call('eth_getBalance', ['0x4838B106FCe9647Bdf1E7877BF73cE8B0BAD5f97', 'latest']);
$balance = Utils::hexToBn($res->result);
echo 'Balance: ' . Utils::fromWeiToString($balance, 'ether') . PHP_EOL;

// Balance: 16.857795621056046838
```

Теперь уже будет больше похоже на правду.

### Генерация аккаунта

Мало смысла в работе с блокчейном, не имея своего собственного адреса, поэтому давайте создадим себе
адрес (публичный и приватный ключ):

```php
use SWeb3\Accounts;

$account = Accounts::create();
echo "Public key: " . $account->publicKey . PHP_EOL;
echo "Private key: " . $account->privateKey . PHP_EOL;
echo "Address: " . $account->address . PHP_EOL;

// Public key: 0x03c484a1c224a35100310228205c40af86a0a511e8ad41ef71470c86b23e25fbfd
// Private key: 0x65e2bda31ccefd2851ef69b268e62a9e00773cfa5506ae5525d3435d4589f2d1
// Address: 0x0e1cd3dc5ceb15f2a515b5cefbcd80ceb935f38d
```


Теперь попробуем получить баланс
[нашего адреса](https://etherscan.io/address/0x0e1cd3dc5ceb15f2a515b5cefbcd80ceb935f38d){:target="_blank"} 
и ожидаемо получим 0:

```php
$res = $node->call('eth_getBalance', [$account->address, 'latest']);
$balance = Utils::hexToBn($res->result);
echo 'Balance: ' . Utils::fromWeiToString($balance, 'ether') . PHP_EOL;

// Balance: 0
```

После того как научились коннектиться к ноде, понимаем как делать к ней запросы и парсить ответы — можно переходить к 
более сложным вещам: отправке ETH и взаимодействию со смарт-контрактами.







