---
title: "MoneyPHP: Работа с деньгами в PHP (Часть 2)"
date: 2025-06-12
categories: [PHP]
tags: [money, php, moneyphp] 
---

## Конвертация валют

### Фиксированный рейт

Когда у нас какой-то интернациональный бизнес, и нужно работать с несколькими валютами, то обязательно встанет вопрос 
конвертации. MoneyPHP из коробки поддерживает консервацию из одной валюты в другую. Для этого используется класс конвертер, 
который создается из валют и их рейтов относительно друг-друга. 

Для простоты рассмотрим примитивный пример, когда мы просто захардкодили отношения между валютами в виде объекта `FixedExchange`:

```php
use Money\Converter;
use Money\Currency;
use Money\Currencies\ISOCurrencies;
use Money\Exchange\FixedExchange;

$exchange = new FixedExchange([
  'EUR' => [
      'USD' => '1.25'
  ]
]);

$converter = new Converter(new ISOCurrencies(), $exchange);

$eur100 = Money::EUR(100);
$usd125 = $converter->convert($eur100, new Currency('USD'));
```

Конечно, в реальном приложении от такого будет мало пользы. Но если это какая-то внутренняя валюта или что-то вроде 
бонусов/баллов — то вполне. 

### Рейт провайдеры

Для более сложных кейсов в MoneyPHP можно использовать интеграции со сторонними провайдерами рейтов. 
Для этого нужно реализовать интерфейс `ExchangeRateProvider`, который для переданной пары base/quote валют отдает рейт:

```php
namespace Exchanger\Contract;

interface ExchangeRateProvider
{
    public function getExchangeRate(ExchangeRateQuery $exchangeQuery): ExchangeRate;
}
```

Дальше такой сервис оборачивается в `ExchangerExchange` и всё. Снова собираем конвертер и обращаемся к нему:

```php
use Money\Money;
use Money\Converter;
use Money\Currencies\ISOCurrencies;
use Money\Exchanger\ExchangerExchange;

// $exchanger = Implementation of ExchangeRateProvider
$exchange = new ExchangerExchange($exchanger);
$converter = new Converter(new ISOCurrencies(), $exchange);

$eurAmount = Money::EUR(100);
$usdAmount = $converter->convert($eurAmount, new Currency('USD'));
```

>Есть отдельные библиотеки, которые реализуют эти интерфейсы с уже готовыми реализациями: 
> [florianv/swap](https://github.com/florianv/swap){:target="_blank"} и [florianv/exchanger](https://github.com/florianv/exchanger){:target="_blank"}
>Единственный минус, что все они для западных площадок и провайдеров. Если нужно что-то вроде Мосбиржи или ЦБ РФ, то
>скорее всего нужно будет писать свою реализацию.
{: .prompt-tip }

С конвертацией валют важно помнить, что нужно обязательно сохранять в базу рейт и его дату. Обязательно будет ситуация, 
когда пожалуется клиент, что ему неверно сконвертили сумму... и нужно будет иметь исторические данные, чтоб понять что, когда
и по какому рейту было посчитано.

![](/assets/img/posts/money-gone.png)

## Форматирование

Еще один челлендж, связанный с валютами, это адаптация пользовательского интерфейса к различным языкам и регионам - проще говоря локализация. 
Сюда мы включаем форматирование чисел и форматирование валют. Представим что у нас есть сумма 12345 долларов. И у нас интернациональный 
бизнес и нужно показывать эту сумму в разных уголках мира, в каждом из которых свои правила:

| Локаль   | Сумма  |
|------------|--------|
|Нидерланды|US$ 12.345,67|
|Польша|12 345,67 USD|
|США|$12,345.67|

Видно, что в разных локалях одна и та же сумма денег показывается совершенно по-разному. Все эти правила форматирования
реализовывать самим будет тот еще квест, поэтому всегда, когда надо показать сумму человеку, используем форматтеры. Из коробки 
в MoneyPHP доступны несколько форматтеров. 

### INTL Форматтер

Если нужно форматирование дробных номиналов и правильно отобразить значок валюты, то можно использовать `IntlMoneyFormatter`:

```php
$money = new Money(1234567, new Currency('USD'));

$currencies = new ISOCurrencies();
$numberFormatter = new \NumberFormatter('nl_NL', \NumberFormatter::CURRENCY);

$moneyFormatter = new IntlMoneyFormatter($numberFormatter, $currencies);

echo $moneyFormatter->format($money); // US$ 12.345,67
```

Под капотом у него будет использоваться расширение `intl`.

### Decimal Форматтер

Если нам не важна локаль и нужно просто привести сумму к децимал строке – используем `DecimalMoneyFormatter`:

```php
$money = new Money(1234567, new Currency('USD'));

$currencies = new ISOCurrencies();
$moneyFormatter = new DecimalMoneyFormatter($currencies);

echo $moneyFormatter->format($money); // outputs 12345.67
```

Ему для работы нужно передать конфиг валют и всё, дальше сразу получаем отформатированное число например для передачи апи.

### AggregateMoneyFormatter

В сложных системах могут быть кейсы, когда нужно использовать разное форматирование для разных валют, например для фиата одно, 
а для крипты – другое. Для этого можно использовать `AggregateMoneyFormatter`, который объединит несколько форматтеров. 
В примере ниже я использую `IntlMoneyFormatter`, чтобы выводить красивое обозначение суммы в нужной локали для фиата (USD) и 
`DecimalMoneyFormatter` для форматирования криптовалют (BTC):

```php
$dollars = new Money(100, new Currency('USD'));
$bitcoin = new Money(100, new Currency('BTC'));

$numberFormatter = new \NumberFormatter('en_US', \NumberFormatter::CURRENCY);
$intlFormatter = new IntlMoneyFormatter($numberFormatter, new ISOCurrencies());
$decimalFormatter = new DecimalMoneyFormatter(new CryptoCurrencies());

$moneyFormatter = new AggregateMoneyFormatter([
    'USD' => $intlFormatter,
    'BTC' => $decimalFormatter,
]);

echo $moneyFormatter->format($dollars); // outputs $1.00
echo $moneyFormatter->format($bitcoin); // outputs 0.0000010
```

## Сериализация

### JSON

Скорее всего наше приложение будет общаться с другими системами по апи, сообщениями или еще как-либо. Тут возникает вопрос: 
"в каком виде передавать объекты денег?". Рассмотрим пример с json-ом. Класс `Money` реализует интерфейс `JsonSerializable`, 
поэтому можно просто передать объект в `json_encode()` и получить результат:

```php
$money = new Money(123, new Currency('USD'));
echo \json_encode($money);
// {"amount":"123","currency":"USD"}
```

Это json представление объекта "как есть": валюта и amount в минимальном возможном номинале. То есть для бакса это 
будут центы, для рубля – копейки. Тут важно договориться заранее со всеми другими частями системы о формате: 
это будет `integer` или `decimal`? Строка с точкой или обязательно целое число. Если это фронтенд, который не будет 
проводить никаких манипуляций с числами, то лучше ему вернуть сразу отформатированное значение с помощью `DecimalMoneyFormatter`:

```php
$currencies = new ISOCurrencies();
$decimalFormatter = new DecimalMoneyFormatter($currencies);

$money = new Money(123, new Currency('USD'));
$moneyJson = [
    'amount' => $decimalFormatter->format($money),
    'currency' => $money->getCurrency(),
];

echo \json_encode($moneyJson);
// output: {"amount":"1.23","currency":"USD"}
```

### Парсинг

В случае когда же наоборот мы читаем данные из внешнего источника – тоже важно знать формат: это будет decimal-строка с точкой 
или целое число в минимальном номинале валюты. Конструктор `Money` принимает `integer`, но если мы получаем decimal — то его 
нужно будет парсить.

Из коробки доступно несколько парсеров: `IntlMoneyParser` и `DecimalMoneyParser`. Скорее всего у себя в приложении
вы будете получать отдельно число и отдельно валюту, поэтому рекомендую использовать `DecimaMoneyParser`:

```php
public function __construct(
    private readonly DecimalMoneyParser $moneyParser
) {
    /* ... */
}

// ...

$jsonStr = '{
    "amountDecimal": "123.45",
    "currency": "USD"
}';
$moneyJson = \json_decode($jsonStr);

$money = $this->decimalMoneyParser->parse(
$moneyJson->amountDecimal,
    new Currency($moneyJson->currency)
);
echo $money->getAmount(); // outputs 12345
```

>При работе с `float`, которые вы получаете извне, важно помнить про приведение типов в PHP. 
> Если нам откуда пришел float меньше чем 0.001, то при приведении к строке, мы получим так называемое представление в e-notation: 
>```php
>$commission = 0.00001; 
>$this->decimalMoneyParser->parse(
>    (string)$commission, new Currency('USD')
>);
>// Cannot parse "1.0E-5" to Money
>``` 
>По умолчанию PHP все значения меньше 0.001 приводит к такой форме. Тут можно использовать `sprintf()` или `number_format()` из PHP.
{: .prompt-warning }


## Кастомные валюты
Что делать, если валют, представленных в библиотеке, нам не хватает? Например, это может быть крипта, 
или какая-то ваша внутренняя игровая или системная валюта.

![](/assets/img/posts/thousand-custom-money.png){: width="500"}

Сначала нужно сообщить библиотеке о нашей новой валюте, чтобы можно было совершать с ней необходимые операции: арифметика, парсинг, форматирование. 
Для этого moneyphp нужно знать только код валюты и минимальный номинал. Можно определить свой собственный CurrencyList:

```php
$shitCoinCurrenciesList = new \Money\Currencies\CurrencyList(
    [
        'PEPE' => 18,
        'BONK' => 18,
        'HMSTR' => 18,
    ]
);

$currencies = new AggregateCurrencies([
    new ISOCurrencies(),
    $shitCoinCurrenciesList
]);

public function __construct(
    private readonly Currencies $currencies
) { }
```

Дальше можно создать общий список валют для приложения через AggregateCurrencies. Передать туда `ISOCurrencies` 
(фиатные валюты из библиотеки) и наш новый список. Дальше везде в коде, где используются валюты будем инжектить интерфейс currencies.

В Symfony например это выглядело бы так через конфиги:

```yaml
Money\Currencies\ISOCurrencies: ~

app.money.shitcoins_list:
  class: 'Money\Currencies\CurrencyList'
    arguments:
      $currencies:
        PEPE: 18
        HMSTR: 18
        BONK: 18

Money\Currencies\Currencies:
  class: 'Money\Currencies\AggregateCurrencies'
    arguments:
      $currencies:
        - '@app.money.shitcoins_list'
        - '@Money\Currencies\ISOCurrencies'
```

Объявляем список валют из библиотеки `Money\Currencies\ISOCurrencies`, потом объявляем наш кастомный "щиткоин список" (`app.money.shitcoins_list`) 
и дальше говорим, что для интерфейса `Currencies` везде инжектим объединение этих двух списков. 
Таким образом у нас все форматтеры и парcеры автоматически подхватят эти новые валюты.

