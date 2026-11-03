# Трансляция событий

Трансляция событий позволяет передавать изменения из приложения в браузер без постоянных HTTP-запросов. 
Для этого используется Centrifugo 6: PHP отправляет задачу в очередь, worker публикует сообщение через HTTP API Centrifugo, а подключённый браузер получает его по WebSocket.

В стартере предусмотрены персональные каналы пользователей. Поддержки публичных каналов и произвольных подписок через API приложения пока нет.

## Настройка Centrifugo

Параметры находятся в `config/settings/broadcast.php`. По умолчанию трансляция отключена. Чтобы её включить, задайте переменные в `.env`:

```dotenv
BROADCAST_ENABLED=true
BROADCAST_API_URL=http://centrifugo:8000
BROADCAST_WEBSOCKET_URL=ws://localhost:8000/connection/websocket
```

Также нужны два отдельных секрета длиной не менее 32 символов:

- `BROADCAST_TOKEN_KEY` используется для подписи клиентских токенов.
- `BROADCAST_API_KEY` используется сервером приложения при обращении к HTTP API Centrifugo.

Docker Compose передаёт эти значения в Centrifugo как `CENTRIFUGO_CLIENT_TOKEN_HMAC_SECRET_KEY` и `CENTRIFUGO_HTTP_API_KEY` соответственно.

Для запуска сервиса и обновления окружения приложения выполните из корня проекта:

```bash
./run --profile realtime up -d centrifugo
./run --profile queue up -d --force-recreate app worker
./run console broadcast:check
```

Команда `broadcast:check` вызывает метод `info` HTTP API и проверяет доступность Centrifugo из PHP-приложения. Она не проверяет подключение браузера по WebSocket.

Разрешённые источники указаны в `docker/centrifugo/config.json`. 
По умолчанию это `http://localhost:5173` и `http://localhost:8080`. 
При изменении адреса фронтенда необходимо обновить список origins и `BROADCAST_WEBSOCKET_URL`. 
Для сайта на HTTPS используйте `wss://`.

## Публикация событий

Для отправки сообщения используется сервис `App\Broadcast\Broadcast`, доступный через DI:

```php
use App\Broadcast\Broadcast;

final class ReportReady
{
    public function __construct(private Broadcast $broadcast) {}

    public function notify(int $userId, string $reportId): void
    {
        $this->broadcast->toUser($userId, 'report.ready', [
            'reportId' => $reportId,
        ]);
    }
}
```

Метод `toUser()` возвращает идентификатор поставленного события или `null`, если трансляция отключена.

Идентификатор пользователя должен быть положительным. 
Имя события начинается с латинской буквы и может содержать до 100 латинских букв, цифр, точек, дефисов и подчёркиваний. 
Данные сообщения должны поддерживать сериализацию в JSON.

Публикация выполняется через `PublishEvent` и `Publisher`. 
Сообщение содержит поля `id`, `event` и `data`, а персональный канал пользователя имеет вид `personal:user_4`.

События об изменении уведомлений публикуются отдельным подписчиком. 
Поскольку отправка происходит через очередь, для доставки необходим работающий worker. П
одробнее этот механизм описан в разделе [Очереди](/docs/{{version}}/queues).

## Подключение фронтенда

Для получения параметров подключения предусмотрен защищённый маршрут `GET /broadcast/connection`. Когда трансляция включена, он возвращает:

```json
{
  "enabled": true,
  "url": "ws://localhost:8000/connection/websocket",
  "channel": "personal:user_42",
  "token": "<JWT>",
  "expiresIn": 300
}
```

Это пример структуры ответа, а не готовый токен. При отключённой трансляции API возвращает только `enabled: false`.

Токен подключения действует 300 секунд. 
В нём указывается пользователь (`sub`) и разрешённый персональный канал.

Во фронтенде подключение реализовано в `frontend/src/services/realtime.ts` с помощью SDK `centrifuge` и composable `useRealtime`. 
Клиент запрашивает и обновляет токен через API, обрабатывает переподключение и позволяет подписываться на события через `realtime.on()`.

`realtime.on()` возвращает функцию отписки. 
Её нужно вызывать, когда подписка больше не требуется, например при уничтожении компонента Vue. 
После восстановления соединения интерфейс повторно запрашивает данные уведомлений через HTTP.