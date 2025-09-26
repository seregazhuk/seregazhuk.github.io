---
title: "Приём крипто-платежей в PHP (часть 2): свипинг"
date: 2025-09-26
categories: [Blockchain, Ethereum, PHP]
tags: [ethereum, php, sign transaction, blockchain, temporal, receive crypto] 
---

После [успешного пополнения invoice-адреса]({% post_url 2025-09-17-receive-crypto-generate-address-and-receive-payment %}){:target="_blank"} 
нужно каким-то образом перевести деньги на один общий счет: hot-wallet или биржу. Процесс перевода 
крипты из разрозненных источников на один какой-то кошелек называется "свипинг". 

Доработаем наш `AcceptCryptoWorkflow` и добавим в него свипинг:

```php
#[WorkflowMethod(name: 'acceptCrypto')]
public function acceptCrypto(AddressWithAmount $request): Generator
{
    $waitingBalance = Workflow::async(
        function () use ($request) {
            while (! yield $this->addressActivity->hasEnoughBalance($request)) {
                yield Workflow::timer(3);
            }
            $this->paymentReceived = true;
        }
    );
    
    yield Workflow::awaitWithTimeout(
        self::WAIT_FOR_PAYMENT_TIMEOUT, fn() => $this->paymentReceived
    );
    if (!$this->paymentReceived) {
        $waitingBalance->cancel();
        // mark order as canceled
    }
    
    // sweep invoice amount
    // mark order as complete
}
```

## SweepingActivity

Сделаем под свипинг отдельный `SweepingActivity`:

```php
#[ActivityInterface]
class SweepingActivity
{
    #[ActivityMethod]
    public function sweep(AddressWithAmount $request): string
    {
        
    }
}
```

Активити будет "свипать" amount из инвойса с адреса из инвойса на наш какой-то общий sweeping-адрес. И 
возвращать хэш свипинг-транзакции:

```php
#[ActivityInterface]
class SweepingActivity
{
    const int ETH_TRANSFER_GAS = 21000;

    public function __construct(
        private readonly SWeb3 $node,
        private readonly BlockchainAddress $sweepingAddress
    )
    {
    }

    #[ActivityMethod]
    public function sweep(AddressWithAmount $request): string
    {
        $this->node->setPersonalData(
            $request->address->address,
            $request->address->getRawPrivateKey()
        );

        $transaction = [
            'from' => $request->address->address,
            'to' => $this->sweepingAddress->address,
            'value' => $request->amount->getAmount(),
            'nonce' => $this->node->personal->getNonce(),
            'gasLimit' => self::ETH_TRANSFER_GAS,
        ];

        $result = $this->node->send($transaction);
        if (isset($result->error)) {
            throw new \RuntimeException($result->error->message);
        }

        return $result->result;
    }
}
```

Далее зарегистрируем новый активити в воркере:

```php
// worker.php
$dotenv = new Dotenv();
$dotenv->load(__DIR__.'/.env');

$factory = WorkerFactory::create();
$worker = $factory->newWorker();

$node = new \SWeb3\SWeb3($_ENV['NODE_ADDRESS']); // https://ethereum-sepolia-rpc.publicnode.com
$node->chainId = $_ENV[$_ENV['CHAIN_ID']]; // Sepolia chain id: 0xaa36a7 
$sweepingAddress = new \App\BlockchainAddress(
    $_ENV['SWEEPING_ADDRESS'],
    $_ENV['SWEEPING_PRIVATE_KEY']
);

$worker->registerActivity(
    AddressActivity::class,
    fn() => new AddressActivity($node)
);
$worker->registerActivity(
    SweepingActivity::class,
    fn() => new SweepingActivity($node, $sweepingAddress)
);

$factory->run();    
```

И последнее – обновить `AcceptCryptoWorkflow`. Добавить и вызвать новый `SweepingActivity`:

```php
#[WorkflowInterface]
class AcceptCryptoWorkflow
{
    public function __construct() {
        $this->addressActivity = Workflow::newActivityStub(
            AddressActivity::class,
            ActivityOptions::new()->withStartToCloseTimeout(10)
        );

        $this->sweepingActivity = Workflow::newActivityStub(
            SweepingActivity::class,
            ActivityOptions::new()->withStartToCloseTimeout(10)
        );
    }

    #[WorkflowMethod(name: 'acceptCrypto')]
    public function acceptCrypto(AddressWithAmount $request): Generator
    {
        $waitingBalance = Workflow::async(
            function () use ($request) {
                while (! yield $this->addressActivity->hasEnoughBalance($request)) {
                    yield Workflow::timer(3);
                }
                $this->paymentReceived = true;
            }
        );
        
        yield Workflow::awaitWithTimeout(
            self::WAIT_FOR_PAYMENT_TIMEOUT, fn() => $this->paymentReceived
        );
        if (!$this->paymentReceived) {
            $waitingBalance->cancel();
            // mark order as canceled
        }

        yield $this->sweepingActivity->sweep($request);
        // mark order complete
    }
}
```

Теперь попробуем запустить и выполнить наш воркфлоу:

>![](/assets/img/posts/receive-crypto-sweeping-failure.png)

Во время свипинга получили ошибку:

```php
insufficient funds for gas * price + value: 
balance 1000000000000000, tx cost 1000021000651000, overshot 21000651000
```


## Комиссия за свипинг

Проблема в том, что за свипинг-транзакцию нужно оплатить газ, но мы свипаем полностью весь баланс адреса. 
В результате на адресе не остается ETH для оплаты комиссии за транзакцию. Как это пофиксить? Можно попробовать 
свипать не весь баланс, а чуть меньше, оставляя немного ETH для комиссии. Это поможет если мы принимаем
платежи в только в нативной валюте блокчейна (ETH). Если же мы работаем с токенами, например USDT, то тут 
уже проблема. На адресе в принципе нет ETH, чтобы оплатить комиссию. Токенами мы газ никак не оплатим. 
Решением этой проблемы будет пополнение invoice-адреса на небольшую сумму ETH, достаточную для покрытия комиссии 
сети. Можно даже совместить два подхода: если инвойс был в нативной валюте (ETH) – свипать не весь баланс, а чуть
меньше. Если инвойс в токенах – пополнять баланс на сумму, необходимую для оплаты комиссии за свипинг.






