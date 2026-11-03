# Middleware

Middleware - это промежуточный обработчик на пути запроса и ответа.
Он может проверить запрос перед следующим шагом, а затем дополнить готовый ответ.
В стартере так устроены CORS, ограничения частоты и проверка авторизации.

## Подключение и порядок выполнения

Глобальные middleware подключаются в `app/Bootstrap/AppFactory.php`, обработчики отдельных маршрутов и групп - в `routes/api.php`.
Slim выполняет добавленные middleware по принципу «последний добавлен - первый вызван».
Из-за этого перестановка строк `add()` меняет порядок обработки запроса.

## Собственный middleware

Для собственного обработчика достаточно реализовать интерфейс PSR-15 `MiddlewareInterface`.
Например, обработчик ниже добавляет заголовок к ответу:

```php
namespace App\Modules\Account\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

final class ResponseSourceMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler,
    ): ResponseInterface {
        $response = $handler->handle($request);

        return $response->withHeader('X-Application', 'Sliku');
    }
}
```

Вызов `$handler->handle($request)` передаёт запрос следующему обработчику. 
Когда тот возвращает ответ, middleware продолжает выполнение и может его изменить. 
При необходимости middleware может завершить обработку раньше и вернуть собственный ответ, не вызывая `handle()`.

Объекты PSR-7 неизменяемы: `withHeader()` возвращает новый экземпляр ответа. Поэтому результат нужно вернуть или сохранить в переменную.

Чтобы подключить middleware к конкретному маршруту, используется `add()`:

```php
->add(ResponseSourceMiddleware::class)
```

## Авторизация и ограничение запросов

`JwtAuthMiddleware` проверяет аутентификацию и делает сведения о текущем пользователе доступными следующим обработчикам. 
`RequireVerifiedEmailMiddleware` ограничивает доступ к операциям, для которых необходимо подтверждение электронной почты.

Ограничения частоты запросов настраиваются в `config/settings/rate_limits.php`.
При превышении лимита приложение возвращает HTTP-статус `429` и заголовок `Retry-After`.

Настройки CORS находятся в `config/settings/cors.php`.