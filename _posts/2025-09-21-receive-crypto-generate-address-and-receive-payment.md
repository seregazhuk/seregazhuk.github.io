---
title: "Приём крипто-платежей в PHP: генерация адреса и проверка баланса"
date: 2025-08-17
categories: [Blockchain, Ethereum, PHP]
tags: [ethereum, php, sign transaction, blockchain, temporal, receive crypto] 
---

## Проблема

Под приёмом крипты можно подразумевать разные кейсы: нам продают крипту за фиат, мы обмениваем крипту на другую крипту, 
мы принимаем крипто-платежи за что-то (услуги или товары). В конечном итоге всё сводится к "получить крипту от пользователя". 
Какие тут могут возникнуть сложности? Проблема в фундаментальном отличии от фиатных платежей. Для оплаты чего-либо в фиате
у нас может быть один банковский счет, на который мы принимаем все входящие платежи. У входящего платежа всегда будет сумма и 
реквизиты отправителя. По этим реквизитам всегда можно сматчить ордер: сумма + отправитель. В крипте же мы подразумеваем, что
все адреса анонимные. Да, на крипто-биржах мы проходим процедуру KYC, даже загружаем свой паспорт, но для стороннего пользователя 
блокчейна никогда не понятно, какому пользователю принадлежит тот или иной кошелек. Всё ещё усложняется тем, что пользователь может
иметь какой свой собственный кошелек (hot wallet), так и например иметь средства на бирже. А биржа может использовать один адрес
для всех исходящих транзакций. Тогда мы у себя будем видеть, что все платежи к нам идут с одного блокчейн-адреса, хотя отправлять их
могут совершенно разные люди. Более того, пользователь может произвести оплату вообще несколькими платежами: один платеж со своего
hot-wallet-а, а второй с биржи. В итоге встаёт вопрос: как имея несколько входящих платежей с разных адресов на разные суммы, понять
что это всё оплата одного ордера?

## Одноразовые адреса

Решение этой проблемы не совсем очевидное, но лежит на поверхности: **нам нужен уникальный блокчейн адрес под каждый ордер**. 
Тогда все эти проблемы с неизвестными адресами отпадают. Мы просто генерируем новый адрес, отдаём его клиенту и потом периодически проверяем его баланс. 
Как только баланс станет равным сумме инвойса считаем, что платеж выполнен.

Попробуем закодить такое решение. Для простоты без фреймворков, только пара библиотек. Без базы данных, просто, чтобы показать сам флоу процесса. 
Единственное, что для долго живущих процессов будем использовать Temporal. В качестве блокчейна будем использовать Ethereum (тестнет Sepolia). 

Пакеты, которые нам понадобятся:
- `temporal/sdk` ([PHP SDK для Temporal](https://github.com/temporalio/sdk-php){:target="_blank"})
- `moneyphp/money` ([библиотека для работы с деньгами](https://github.com/moneyphp/money){:target="_blank"})
- `symfony/dotenv` ([чтение env-файлов](https://github.com/symfony/dotenv){:target="_blank"})
- `drlecks/simple-web3-php` ([библиотека для работы с Ethereum](https://github.com/drlecks/Simple-Web3-Php){target="_blank"})

Итак, начнем с генерации адреса для инвойса. Представим, что у нас инвойс на 0.001 ETH:

```php

$amount = \Money\Money::ETH('1000000000000000'); // 0.001 ETH
$invoiceAccount = \SWeb3\Accounts::create(); 
$invoiceAddress = new BlockchainAddress(
    $invoiceAccount->address,
    $invoiceAccount->privateKey
);

echo "Invoice address: $invoiceAddress->address" . PHP_EOL;
echo "Invoice amount: 0.001 ETH" . PHP_EOL;
```

Нам дальше постоянно нужно будет использовать связку адрес + приватный ключ, поэтому сделаем под это отдельный класс DTO:
```php
final readonly class BlockchainAddress
{
    public function __construct(public string $address, public string $privateKey)
    {
    }

    public function getRawPrivateKey(): string
    {
        return str_replace('0x', '', $this->privateKey);
    }
}
```

Теперь, сколько бы мы не запускали этот скрипт, всегда будет генерироваться новый адрес. Дальше нам нужно принять крипту на него.

## Проверка баланса адреса

Проверка баланса сгенерированного адреса с определенной периодичностью – подходящая задача для Temporal. Создадим Workflow для приема крипты:

```php

use App\AddressWithAmount;
use Generator;
use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\WorkflowMethod;

#[WorkflowInterface]
class AcceptCryptoWorkflow
{
    #[WorkflowMethod(name: 'acceptCrypto')]
    public function acceptCrypto(AddressWithAmount $request): Generator
    {
       
    }
}
```

Мы постоянно будем использовать пару address + amount, поэтому создадим для него DTO:

```php
class AddressWithAmount
{
    public function __construct(
        public BlockchainAddress $address,
        #[Marshal(type: MoneyType::class)]
        public Money $amount,
    ) {
    }
}
```

>Класс `MoneyType` нужен, чтобы показать Temporal, каким образом нужно передавать объект `Money` из активити в воркфлоу (сериализовать и парсить) и наборот:
>```php
>use Money\Currency;
>use Money\Money;
>use Temporal\Internal\Marshaller\Type\Type;
>
>/**
>  * @extends Type<array>
>  */
>class MoneyType extends Type
>{
>    /**
>     * @param mixed|Money $value
>     */
>    public function serialize($value): array
>    {
>        return [
>            'amount' => $value->getAmount(),
>            'currency' => $value->getCurrency()->getCode(),
>        ];
>    }
>
>    /**
>     * @param mixed|array $value
>     * @param mixed       $current
>     */
>    public function parse($value, $current): Money
>    {
>        return new Money(
>            $value['amount'],
>            new Currency($value['currency'])
>        );
>    }
>}
>```
{: .prompt-info }

Воркфлоу пока оставим пустым, создадим активити:

```php
#[ActivityInterface]
readonly class AddressActivity
{
    public function __construct(private SWeb3 $node) {
    }

    #[ActivityMethod]
    public function hasEnoughBalance(AddressWithAmount $request): bool
    {
        $rawBalance = $this->node->call(
            'eth_getBalance',
            [$request->address->address, 'latest']
        );
        $balance = Money::ETH(Utils::hexToBn($rawBalance->result)->toString());
        return $balance->greaterThanOrEqual($request->amount);
    }
}
```

Активити `AddressActivity` проверяет, что на балансе переданного адреса достаточно ETH. 

>Для простоты примера будем проверять только баланс нативной валюте. Если бы работали и с токенами – то пример был бы сложнее, и 
>нужно было бы [вызывать метод смарт-контракта]({% post_url 2025-04-04-send-erc20-php %}){:target="_blank"}).
{: .prompt-tip }

Теперь вернемся в воркфлоу и допишем недостающий код:

```php
use App\Activity\AddressActivity;
use App\AddressWithAmount;
use App\RefillAmount;
use Generator;
use Temporal\Activity\ActivityOptions;
use Temporal\Workflow;
use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\WorkflowMethod;

#[WorkflowInterface]
class AcceptCryptoWorkflow
{
    public function __construct() {
        $this->addressActivity = Workflow::newActivityStub(
            AddressActivity::class,
            ActivityOptions::new()->withStartToCloseTimeout(10)
        );
    }

    #[WorkflowMethod(name: 'acceptCrypto')]
    public function acceptCrypto(AddressWithAmount $request): Generator
    {
        while (! yield $this->addressActivity->hasEnoughBalance($request)) {
            yield Workflow::timer(3);
        }

        // mark order as complete
    }
}
```

В воркфлоу выше вызываем активити, до тех пор пока баланс на адресе будет больше или равен сумме из ордера. Если баланса
не достаточно – ждём 3 секунды.

И финальная часть – зарегистрировать наши воркфлоу и активити в воркере RoadRunner-а. В качестве блокчейн ноды будем использовать
[бесплатную публичную Sepolia ноду](https://ethereum-sepolia-rpc.publicnode.com){:target="_blank"}:

```php
$factory = WorkerFactory::create();
$worker = $factory->newWorker();

$node = new \SWeb3\SWeb3('https://ethereum-sepolia-rpc.publicnode.com');
$worker->registerActivity(AddressActivity::class, fn() => new AddressActivity($node));
$worker->registerWorkflowTypes(AcceptCryptoWorkflow::class);

$factory->run();
```

## Запуск воркфлоу для получения крипто-платежа

Теперь если мы сгенерируем блокчейн-адрес и запустим с ним воркфлоу, то после отправки нужной суммы ETH на него – воркфлоу будет 
выполнен:

```php
use App\BlockchainAddress;
use Temporal\Client\GRPC\ServiceClient;
use Temporal\Client\WorkflowClient;

$workflowClient = WorkflowClient::create(
    serviceClient: ServiceClient::create('127.0.0.1:7233')
);

$amount = \Money\Money::ETH('1000000000000000'); // 0.001 ETH
$invoiceAccount = \SWeb3\Accounts::create(); 
$invoiceAddress = new BlockchainAddress(
    $invoiceAccount->address,
    $invoiceAccount->privateKey
);

echo "Invoice address: $invoiceAddress->address" . PHP_EOL;
echo "Invoice amount: 0.001 ETH" . PHP_EOL;

$workflow = $workflowClient->newWorkflowStub(\App\Workflow\AcceptCryptoWorkflow::class);
$request = new \App\AddressWithAmount($invoiceAddress, $amount);
$workflowClient->start($workflow, $request);
```

