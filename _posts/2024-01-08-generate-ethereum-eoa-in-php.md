---
title: Генерируем Ethereum EOA адреса в PHP 
date: 2024-01-08
categories: [Blockchain, Ethereum]
tags: [ethereum, address generation]     # TAG names should always be lowercase
---

### Что такое адрес в Ethereum?

Адрес представляет собой 20-байтовое шестнадцатеричное число, которое используется для идентификации аккаунта в блокчейне Ethereum. Адрес - это уникальный идентификатор, который используется для отправки, получения и хранения Eth, токенов, а так же доступа к децентрализованным приложениям. Любой адрес состоит из строки буквенно-цифровых символов и обычно начинается с "0x", что указывает на его шестнадцатеричный формат.

### Типы аккаунтов (адресов)

В блокчейне Ethereum есть два типа аккаунтов и соответственно у каждого свой адрес:
-  **Externally owned account (EAC)** - аккаунт под управлением реального пользователя. EOA в основном используются для инициирования транзакций, таких как отправка эфира или токенов на другие адреса. EOA можно создать, создав новую учетную запись Ethereum с помощью кошелька (например MetaMask).
- Contract account - принадлежат смарт-контрактам и могут использоваться для взаимодействия с блокчейном Ethereum. Адреса контрактов — это уникальные адреса, которые связаны со смарт-контрактами, развернутыми на блокчейне Ethereum.

#### Отличия между EOA и Contract Account

- **Создание:** EOA создаются пользователями. Contract account создаются при деплое смарт-контракта.
- **Key Pair:** EOA имеют пару публичный-приватный ключ. Приватный ключ используется для подписи транзакций. У контракта же наоборот нет ни приватного, ни публичного ключа.
- **Управление:** пользователи имеют контроль над закрытыми ключами, связанными с их EOA. Contract account контролируются логикой кода смарт-контракта. Код определяет правила и поведение аккаунта контракта.
- **Подпись транзакций:** Только EOA могут подписывать транзакции, поскольку у них есть закрытый ключ. Подпись, сгенерированная с использованием закрытого ключа, гарантирует подлинность и целостность транзакции. Аккаунты контрактов не могут подписывать транзакции, поскольку у них нет своего закрытого ключа.

#### Сходства EOA и Contract Account

- Имеют адреса, представленные в виде 20-байтовых шестнадцатеричных строк, которые идентифицируют аккаунт в блокчейне Ethereum.
- Могут хранить как Ether, так и токены ERC-20.
- Могут отправлять и получать Ether, и взаимодействовать с децентрализованными приложениями (DApps).

### Криптография адресов в Ethereum

Адреса в блокчейне Ethereum генерируются с помощью алгоритма ECDSA (Elliptic Curve Digital Signature Algorithm). ECDSA — это криптографический алгоритм, который использует пару ключей: открытый ключ и закрытый ключ, для подписания и проверки цифровых подписей. Это значит что сгенерировать адрес мы всегда можем "локально", взаимодействия с сетью не нужно.

Эллиптическая кривая в ECDSA — это линия на плоскости, задаваемая уравнением `y²=x³+a∙x+b`. В Bitcoin и Ethereum используют кривую `y²=x³+7`, которая называется `sepc256k1`:
![Эллиптическая кривая sepc256k1](/assets/img/posts/ethereum-elliptic-curve.png)
_Эллиптическая кривая sepc256k1_

Открытый ключ вычисляется из закрытого ключа с помощью "умножения эллиптической кривой": `K = k * G`, где `k` — закрытый ключ, `G` — константная точка, называемая точкой генератора, `K` — получающийся открытый ключ, а `*` — специальный оператор «умножения» эллиптической кривой.
Обратите внимание, что умножение эллиптической кривой не похоже на обычное умножение.

Арифметика на эллиптической кривой отличается от «обычной» целочисленной арифметики. Точка (G) может быть умножена на целое число (k), чтобы получить другую точку (K). Но такого понятия, как деление, не существует, поэтому невозможно просто «разделить» открытый ключ K на точку G, чтобы вычислить закрытый ключ k. Это односторонняя математическая функция.
Закрытый ключ можно преобразовать в открытый ключ, но открытый ключ нельзя преобразовать обратно в закрытый ключ, потому что математика работает только в одну сторону. Закрытый ключ используется для подписи транзакций и подтверждения права собственности на адрес.

Существует множество эллиптических кривых для различных типов криптовалют, которые широко используются в мире. Ethereum использует кривую secp256k1.

### Как сгенерировать EOA адрес

Итак, первое, что нам нужно - сгенерировать приватный ключ. Для этого можно использовать OpenSSL функции:

```php
$privateKey = openssl_pkey_new([  
    'private_key_type' => OPENSSL_KEYTYPE_EC,  
    'curve_name' => 'secp256k1'  
]);
```

Через конфиг указываем явно, что нам нужен ключ типа "эллиптическая кривая". В качестве самой кривой указываем используемую в Ethereum secp256k1.

Далее нужно получить строковое представление ключа:

```php
openssl_pkey_export($privateKey, $privateKeyString);
```

Так мы получим PEM представление приватного ключа. Если вывести содержимое переменной `$privateKeyString`, то там будет  следующее:

```text
-----BEGIN PRIVATE KEY-----
MIGEAgEAMBAGByqGSM49AgEGBSuBBAAKBG0wawIBAQQgy9ZX4bAPGTmPePsvuJKE
v9O2LGi1p7XPm9Kz3Lvby+2hRANCAASE5DEX2+oI8aD3bWyh9IZXyjMil05V6LD1
CHqytRTTjAMFBP3k2CyD91Y9xhHz+yXiEVUlrODyOT1mqfuTNedH
-----END PRIVATE KEY-----
```

В PEM-формате хранится Base64-закодированное бинарное представление приватного ключа.

Прежде чем продолжить дальше установим необходимые библиотеки:

```bash
composer req sop/crypto-encoding sop/crypto-types kornrunner/keccak
```

- `sop/crypto-encoding` — для работы с PEM-файлами.
- `sop/crypto-types` — для работы с ASN.1 структурами.
- `kornrunner/keccak` — для работы с _Keccak_ хэшами.

Теперь нужно перевести PEM приватный ключ в формат Elliptic Curve:

```php
$privatePem = PEM::fromString($privateKeyString);  
$ecPrivateKey = ECPrivateKey::fromPEM($privatePem);
```

Обычно EC приватный ключ представляется в виде ASN.1 (Abstract Syntax Notation One):

```text
ECPrivateKey ::= SEQUENCE {
     version        INTEGER { ecPrivkeyVer1(1) } (ecPrivkeyVer1),
     privateKey     OCTET STRING,
     parameters [0] ECParameters {{ NamedCurve }} OPTIONAL,
     publicKey  [1] BIT STRING OPTIONAL
   }
```

В нашем конкретном примере структура будет выглядеть так:

```text
SEQUENCE
  INTEGER 00
  SEQUENCE
    ObjectIdentifier ecPublicKey (1 2 840 10045 2 1)
    ObjectIdentifier secp256k1 (1 3 132 0 10)
  OCTETSTRING, encapsulates
    SEQUENCE
      INTEGER 01
      OCTETSTRING cbd657e1b00f19398f78fb2fb89284bfd3b62c68b5a7b5cf9bd2b3dcbbdbcbed
      [1]
        BITSTRING 000484e43117dbea08f1a0f76d6ca1f4..(total 66bytes)..e2115525ace0f2393d66a9fb9335e747
```

Из этой структуры нам нужно достать приватный и публичный ключи (1-ый `OCTETSTRING` и 3-ий `BITSTRING` элемент):

```php
$privKeyHex = bin2hex($ecPrivateKeyInfo->at(1)->asOctetString()->string());  
$pubKeyHex = bin2hex(
    $ecPrivateKeyInfo->at(3)->asTagged()->asExplicit()->asBitString()->string()
);
```

Приватный и публичный ключи у нас есть, теперь нужно из EC публичного ключа получить адрес Ethereum. Для этого нужно:
- получить Keccak-256 хэш от публичного ключа;
- из полученного хэша взять последние 20 байт;
- добавить в начало `0x`, чтобы явно указать что это 16-ый формат;

Любой EC публичный ключ всегда начинается с последовательности `04`, поэтому перед хэшированием удаляем её. Получаем Keccak256 хэш. Адрес в сети Ethereum всегда 20 байт, что включает в себя 40 символов в длину. Поэтому из полученного хэша берем последние 40 символов и добавляем в начало `0x`. Полученная строка и будет адресом:

```php
$trimmedPubKeyHex = substr($pubKeyHex, 2);  
$hash = Keccak::hash(hex2bin($trimmedPubKeyHex), 256);
$address = '0x' . substr($hash, -40);
```

В результате получаем пару "приватный/публичный ключ" и адрес:

```php
echo sprintf(  
    "Address: %s\nPrivate key: 0x%s\nPublic key: 0x%s\n",  
    $address, $privKeyHex, $pubKeyHex  
);
``` 

```bash
Address: 0x0c20c90899521822b77bd317e45681e2ab5b3b54
Private key: 0xcbd657e1b00f19398f78fb2fb89284bfd3b62c68b5a7b5cf9bd2b3dcbbdbcbed
Public key: 0x0484e43117dbea08f1a0f76d6ca1f48657ca3322974e55e8b0f5087ab2b514d38c030504fde4d82c83f7563dc611f3fb25e2115525ace0f2393d66a9fb9335e747
```

Если коротко, **алгоритм генерации EOA адреса** в Ethereum:
1. Выбираем эллиптическую кривую: `secp256k1`.
2. Генерируем случайно приватный ключ (мы использовали OpenSSL).
3. Вычисляем публичный ключ, используя закрытый ключ и уравнение эллиптической кривой.
4. Хэшируем публичный ключ с помощью `Keccak-256`.
5. Берем последние 20 байтов хэша.
6. Добавляем префикс `"0x"`.

```php  
use Sop\CryptoTypes\Asymmetric\EC\ECPrivateKey;  
use Sop\CryptoEncoding\PEM;  
use kornrunner\Keccak;

// Generate PEM key
$privateKey = openssl_pkey_new([  
    'private_key_type' => OPENSSL_KEYTYPE_EC,  
    'curve_name' => 'secp256k1'  
]);  
openssl_pkey_export($privateKey, $privateKeyString);  

// Create EC key from PEM
$privatePem = PEM::fromString($privateKeyString);  
$ecPrivateKey = ECPrivateKey::fromPEM($privatePem);  
$ecPrivateKeyInfo = $ecPrivateKey->toASN1();  

// Extract private/public keys from ASN.1
$privKeyHex = bin2hex($ecPrivateKeyInfo->at(1)->asOctetString()->string());  
$pubKeyHex = bin2hex(
    $ecPrivateKeyInfo->at(3)->asTagged()->asExplicit()->asBitString()->string()
);  

// Generate address from public key
$trimmedPubKeyHex = substr($pubKeyHex,2);  
$hash = Keccak::hash(hex2bin($trimmedPubKeyHex), 256);  
$address = '0x' . substr($hash, -40);  

echo sprintf(  
    "Address: %s\nPrivate key: 0x%s\nPublic key: 0x%s\n",  
    $address, $privKeyHex, $pubKeyHex  
);  
```

### Лайфхак парсинга PEM-ключа

На самом деле в PEM-формате хранится Base64-закодированное бинарное представление приватного ключа. Поэтому можно  попробовать получить приватный и публичный ключи прямо из PEM-строки "в лоб" без установки дополнительных библиотек.

Для начала уберем строки вида `-----BEGIN PRIVATE KEY-----` и `-----END PRIVATE KEY-----` и получим Base64-закодированную бинарную строку. Попробуем привести её к шестнадцатиричному виду:

```php
$privateKey = openssl_pkey_new([  
    'private_key_type' => OPENSSL_KEYTYPE_EC,  
    'ec' => ['curve_name' => 'secp256k1'],  
]);  
openssl_pkey_export($privateKey, $privateKeyString);  
  
$base64 = trim(str_replace(  
    ["-----BEGIN PRIVATE KEY-----\n", "\n-----END PRIVATE KEY-----"],  
    '',  
    $privateKeyString  
));
$asn1Hex = bin2hex(base64_decode($base64, true));
```

В результате получаем hex-строку с EC приватным ключом в ASN.1 (Abstract Syntax Notation One) формате:

```text
308184020100301006072a8648ce3d020106052b8104000a046d306b0201010420cbd657e1b00f19398f78fb2fb89284bfd3b62c68b5a7b5cf9bd2b3dcbbdbcbeda1440342000484e43117dbea08f1a0f76d6ca1f48657ca3322974e55e8b0f5087ab2b514d38c030504fde4d82c83f7563dc611f3fb25e2115525ace0f2393d66a9fb9335e747
```

И как мы видели выше первая часть этой строки всегда будет одинаковая (описывает, что это ключ кривой `secp256k1`), а другая часть уже содержит в себе приватный и публичный ключи:

>308184020100301006072a8648ce3d020106052b8104000a046d306b0201010420**cbd657e1b00f19398f78fb2fb89284bfd3b62c68b5a7b5cf9bd2b3dcbbdbcbed**a144034200**0484e43117dbea08f1a0f76d6ca1f48657ca3322974e55e8b0f5087ab2b514d38c030504fde4d82c83f7563dc611f3fb25e2115525ace0f2393d66a9fb9335e747**

Всё, что не выделено — просто служебная информация и для всех сгенерированных `secp256k1` она по идее будет одинаковой. Поэтому ключи можно просто вырезать регуляркой:


```php
$asn1Hex = bin2hex(base64_decode($base64, true));  
$template = '~308184020100301006072a8648ce3d020106052b8104000a046d306b0201010420(\w+)a144034200(\w+)~';  
preg_match($template, $asn1Hex, $matches);
```

В `$matches[1]` будет приватный ключ, а в `$matches[2]` — публичный. 

## Библиотеки
Разбираться во всех этих хэшах и структурах данных конечно очень ~~не~~интересно, но что если хочется готового решения? Есть библиотека [kornrunner/php-ethereum-address](https://github.com/kornrunner/php-ethereum-address):

```bash
composer require kornrunner/ethereum-address
```

Где сразу из коробки можно как сгенерировать новый адрес, так загрузить из приватного ключа:

```php
use kornrunner\Ethereum\Address;

$address = new Address();
// or $address = new Address($privateKey);

echo sprintf(  
    "Address: %s\nPrivate key: 0x%s\nPublic key: 0x%s\n",  
    $address->get(),  
    $address->getPrivateKey(),  
    $address->getPublicKey()  
) ;
```
