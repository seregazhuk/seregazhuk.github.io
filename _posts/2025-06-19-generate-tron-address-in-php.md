---
title: Генерируем Tron адрес в PHP 
date: 2025-06-19
categories: [Blockchain, Tron]
tags: [Tron, address generation, blockchain] 
---

Tron использует account-модель адреса. Адрес — это уникальный идентификатор аккаунта, для работы с которым требуется 
подпись закрытого ключа. Аккаунт имеет множество атрибутов: балансы, bandwidth, energy и т. д. Аккаунт может отправлять
Аккаунты являются основой всех действий в блокчейне Tron.

## Зависимости

Для генерации Tron адреса нам нужно будет вычислять Keccak-256 хэш и кодировать данные в Base58:

```bash
composer req kornrunner/keccak stephenhill/base58 
```

## Алгоритм

### Пара ключей 

Для генерации адреса нужна будет пара приватный/публичный ключ. В Tron-е для подписи транзакций используется кривая _SECP256K1_. 
Генерацию такой пары мы уже [рассматривали на примере Ethereum]({% post_url 2025-01-08-generate-ethereum-eoa-in-php %}). 
Для Tron-а алгоритм будет точно такой же, код можно взять от туда.

### SHA3

Хэшируем публичный ключ (без `0x`) в Keccak-256 (sha3–256):

```php
// $pubKeyHex = '04e824786429af0135e2c4636a206467b5def515a8bc74d7f0b524ca9cf5078db9c3c46cfd1a44d294db9dd19d4ead589cad5002447659cbf6c14486b9cd75ac84' 

$pubKeyBin = hex2bin($pubKeyHex);
$hash = Keccak::hash($pubKeyBin, 256);
$addressHex = '41' . substr($hash, -20);
```

>Все хэши берутся от двоичных данных, не от строк в 16-ой системе!
{: .prompt-info }

Из полученного хэша берем последние 20 байт и спереди добавляем `41`. Длина адреса сейчас должна быть 21 байт.

### Base58Check

Дважды хэшируем в sha256 полученный адрес и берём первые 4 байта из второго хэша в качестве чексуммы:

```php
$addressBin = hex2bin($addressHex);
$hash0 = hash('sha256', $addressBin, true); 
$hash1 = hash('sha256', $hash0, true);
$checksum = substr($hash1, 0, 4);
```

Добавляем чексумму в конец адреса, кодируем это в Base58 и получаем адрес в формате base58check:

```php
$base58 = new StephenHill\Base58('123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz');
$addressBase58 = $base58->encode($checksum);
echo $addressBase58 . PHP_EOL;
```

Полученный адрес всегда начинается с `T` и имеет 34 байта в длину. 

Проверить себя можно [здесь](https://secretscan.org/PrivateKeyTron){:target="_blank"}. Вставить приватный ключ (без `0x`) и увидеть по шагам, как
генерируется адрес. 

![](/assets/img/posts/tron-address-generator-check.png)

Например, здесь из приватного ключа: 

```
0xc79d712296aa5ce7f386b0844ce474e5135451077c92678c6c64cc0d2b811028
``` 

был получен `TTwpCeW31tAxmVsoC6ezWnRtXNg99LiELE` адрес.

## Активация адреса

Чтобы активировать новый адрес в сети Tron нужно с уже существующего аккаунта вызвать любой из 3-ех API:
- напрямую вызвать [Create Account API](https://developers.tron.network/reference/account-createaccount){:target="_blank"}
- перевести TRX на новый адрес
- перевести TRC10 токены на новый адрес. 

>Важно: перевод TRC20 токенов не активирует новый адрес.
{: .prompt-warning }

После подтверждения транзакции на блокчейне можно будет запрашивать информацию о новом адресе. Создание аккаунта 
сжигает у создателя или накопленный bandwidth, или 0.1 TRX (если bandwidth недостаточно).



