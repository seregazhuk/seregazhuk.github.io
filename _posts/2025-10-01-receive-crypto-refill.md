---
title: "Приём крипто-платежей в PHP (часть 3): пополнение адреса"
date: 2025-10-01
categories: [Blockchain, Ethereum, PHP]
tags: [ethereum, php, sign transaction, blockchain, temporal, receive crypto] 
---

[Попытка свипнуть весь баланс]({% post_url 2025-09-26-receive-crypto-sweeping %}){:target="_blank"} 
под ноль будет всегда фэйлится, так как не хватит средств для оплаты комиссии за транзакцию. 
Поэтому перед свипингом надо обязательно пополнить адрес на сумму ETH, достаточную для покрытия
комиссии:

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

    // make refill
    yield $this->sweepingActivity->sweep($request);
    // mark order complete
}
```

## Считаем нужную сумму
Для начала нужно понять на сколько необходимо пополнить адрес из инвойса. Мы знаем, что "стоимость" транзакции
измеряется в газе. А газ уже в свою очередь имеет цену в ETH. Тогда сумма для пополнения будет равна:

```
GAS * GAS_PRICE * MULTIPLIER
```

Нужно получить кол-во газа, необходимое для отправки транзакции. Получить текущую цену газа в ETH. Перемножить эти два
числа. Плюс, нужно учесть, что цена газа не постоянная и может меняться. Цена может пойти вниз, пока мы подписываем
и бродкастим транзакцию. Тогда мы переведем чуть больше, чем нужно. Но также цена может пойти и вверх. И тогда мы переведем 
меньше и свипинг-транзакция снова застрянет. Поэтому лучше подстраховаться и добавить какой-то свой внутренний 
multiplier, на который будем итоговую сумму умножать. Для тестнета можно использовать например `4` или `5`. Для 
мейннета будет достаточно увеличить сумму на 20% (умножить на `1.2`).

В `SweepingActivity` добавим новый метод, который будет считать сумму, необходимую для рефила:

```php
#[ActivityInterface]
class SweepingActivity
{
    private const int REFILL_AMOUNT_MULTIPLIER = 4;
    
    // ... 
    
    #[ActivityMethod]
    public function calcRefillAmount(): RefillAmount
    {
        $rawGasPrice = $this->node->call('eth_gasPrice')->result;
        $gasPrice = Money::ETH(Utils::hexToBn($rawGasPrice)->toString());
        $refillAmount = $gasPrice
            ->multiply(self::ETH_TRANSFER_GAS)
            ->multiply(self::REFILL_AMOUNT_MULTIPLIER);

        return new RefillAmount($refillAmount);
    }
}
```

## Пополняем
Далее добавим метод для пополнения адреса. Для простоты будем использовать наш свипинг адрес для пополнения: 

```php
#[ActivityMethod]
public function refill(AddressWithAmount $request): string
{
    $this->node->setPersonalData(
        $this->sweepingAddress->address,
        $this->sweepingAddress->getRawPrivateKey()
    );

    $transaction = [
        'from' => $this->sweepingAddress->address,
        'to' => $request->address->address,
        'value' => $request->amount->getAmount(),
        'nonce' => $this->node->personal->getNonce(),
        'gasLimit' => self::ETH_TRANSFER_GAS,
    ];

    $result = $this->node->send($transaction);
    if (isset($result->error)) {
        throw new \RuntimeException($result->error->message);
    }

    return $result->result;
```

Метод очень похож на метод, где мы свипаем. Только здесь мы подписываем транзакцию свипинг-кошельком и делаем
транзакцию на адрес из инвойса. И возвращаем хэш рефил-транзакции.

## Финальный флоу

Вернёмся и доработаем воркфлоу:

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

    /** @var RefillAmount $refillAmount */
    $refillAmount = yield $this->sweepingActivity->calcRefillAmount();
    $refillRequest = new AddressWithAmount($request->address, $refillAmount->amount);
    yield $this->sweepingActivity->refill($refillRequest);

    yield $this->sweepingActivity->sweep($request);

```

Перед свипингом добавим пополнение адреса. И в принципе это уже более-менее готовое решение. Единственное, что
скорее всего свипинг сначала упадёт, а уже на 2-ой или 3-ий ретрай сработает. Дело в том, что после пополнения 
должно пройти некоторое время, пока refill-транзакция попадет в блокчейн и будет подтверждена. Поэтому перед 
свипингом лучше сначала немного подождать, пока баланс инвойс-адреса не станет равным сумме ивнойса + сумма пополнения:

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

    /** @var RefillAmount $refillAmount */
    $refillAmount = yield $this->sweepingActivity->calcRefillAmount();
    $refillRequest = new AddressWithAmount($request->address, $refillAmount->amount);
    yield $this->sweepingActivity->refill($refillRequest);

    $balanceWithRefill = $request->amount->add($refillAmount->amount);
    $balanceCheckRequest = new AddressWithAmount($request->address, $balanceWithRefill);
    while (! yield $this->addressActivity->hasEnoughBalance($balanceCheckRequest)) {
        yield Workflow::timer(3);
    }

    yield $this->sweepingActivity->sweep($request);
}
```

Вот это уже финальный рабочий флоу:
1. Ждем пока баланс адреса из инвойса не будет больше или равен сумме инвойса.
2. Считаем сумму, на которую нужно пополнить адрес, чтобы покрыть комиссию за свипинг.
3. Пополняем адрес.
4. Ждём изменения баланса на адресе.
5. Свипаем.

Дальше уже в зависимости от вашего бизнеса, можно либо хэджировать крипту, продавая её на бирже. Или использовать
полученные средства дальше в обороте.

Код с рабочим примером доступен на [GitHub](https://github.com/seregazhuk/receive-crypto-live-coding-demo){:target="_blank"}.







