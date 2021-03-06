Ограничение частоты запросов
===============================

Чтобы избежать злоупотреблений, вам следует подумать о добавлении ограничения частоты запросов к вашим API. Например,
вы можете ограничить использование API до 100 вызовов в течение 10 минут для каждого пользователя. Если от пользователя
в течение этого периода времени приходит большее количество запросов, будет возвращаться ответ с кодом состояния 429
(«слишком много запросов»).

Чтобы включить ограничение частоты запросов, *[[yii\web\User::identityClass|класс user identity]]* должен реализовывать
интерфейс [[yii\filters\RateLimitInterface]]. Этот интерфейс требует реализации следующих трех методов:

* `getRateLimit()`: возвращает максимальное количество разрешенных запросов и период времени, например `[100, 600]`, что
  означает не более 100 вызовов API в течение 600 секунд.
* `loadAllowance()`: возвращает оставшееся количество разрешенных запросов и *UNIX-timestamp* последней проверки
  ограничения.
* `saveAllowance()`: сохраняет оставшееся количество разрешенных запросов и текущий *UNIX-timestamp*.

Вы можете использовать два столбца в таблице user для хранения количества разрешённых запросов и времени последней проверки.
В методах `loadAllowance()` и `saveAllowance()` можно реализовать чтение и сохранение значений этих столбцов в соответствии
с данными текущего аутентифицированного пользователя. Для улучшения производительности можно попробовать хранить эту
информацию в кэше или NoSQL хранилище.

Реализация в модели `User` может быть, например, такой:

```php
public function getRateLimit($request, $action)
{
    return [$this->rateLimit, 1]; // $rateLimit запросов в секунду
}

public function loadAllowance($request, $action)
{
    return [$this->allowance, $this->allowance_updated_at];
}

public function saveAllowance($request, $action, $allowance, $timestamp)
{
    $this->allowance = $allowance;
    $this->allowance_updated_at = $timestamp;
    $this->save();
}
```


Как только соответствующий интерфейс будет реализован в классе identity, Yii начнёт автоматически проверять ограничения
частоты запросов при помощи [[yii\filters\RateLimiter]], фильтра действий для [[yii\rest\Controller]]. При превышении
ограничений будет выброшено исключение [[yii\web\TooManyRequestsHttpException]].

Вы можете настроить ограничитель частоты запросов в ваших классах REST-контроллеров следующим образом:

```php
public function behaviors()
{
    $behaviors = parent::behaviors();
    $behaviors['rateLimiter']['enableRateLimitHeaders'] = false;
    return $behaviors;
}
```

При включенном ограничении частоты запросов каждый ответ, по умолчанию, возвращается со следующими HTTP-заголовками,
содержащими информацию о текущих ограничениях:

* `X-Rate-Limit-Limit`: максимальное количество запросов, разрешённое в течение периода времени;
* `X-Rate-Limit-Remaining`: оставшееся количество разрешённых запросов в текущем периоде времени;
* `X-Rate-Limit-Reset`: количество секунд, которое нужно подождать до получения максимального количества разрешённых
  запросов.

Вы можете отключить эти заголовки, установив свойство [[yii\filters\RateLimiter::enableRateLimitHeaders]] в `false`,
как показано в примере кода выше.
