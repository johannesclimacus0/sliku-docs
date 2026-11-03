# Кэш

В `App\Cache\Cache` можно положить результат вычисления или небольшие служебные данные, чтобы не получать их заново при каждом запросе.
Внутри используются PSR-6 и Symfony Cache, а данные по умолчанию лежат в Redis.
Например, можно на несколько минут сохранить небольшой справочник или результат тяжелого запроса.

## Подключение Redis

Настройки находятся в `config/settings/cache.php`, создание соединения и пула - в `config/definitions/cache.php`:

```dotenv
REDIS_URL=redis://redis:6379
CACHE_NAMESPACE=app
```

Если несколько приложений или окружений используют один Redis, задайте каждому свой namespace.
Стандартное соединение создается с таймаутами подключения и чтения по две секунды.
Если включено профилирование, пул может быть обернут в `TraceableAdapter`.
В вашем коде при этом используется тот же сервис.

`Cache` можно сразу передать в конструктор: отдельно регистрировать каждый класс, которому нужен кэш, не придется.
Вот как читать и записывать значения через сервис:

```php
use App\Cache\Cache;

final class ExportStatus
{
    public function __construct(private Cache $cache) {}

    public function start(int $userId): void
    {
        $this->cache->set('export_'.$userId, ['status' => 'queued'], 600);
    }

    public function read(int $userId): array
    {
        return $this->cache->get('export_'.$userId, ['status' => 'unknown']);
    }
}
```

## Время жизни и вычисление значения

Сигнатура записи - `set(string $key, mixed $value, int|DateInterval|null $ttl = null): bool`.
Целое число означает секунды, `DateInterval` - относительный интервал, `null` не задает срок истечения.
Сервис передает TTL в `expiresAfter()`, поэтому не используйте Unix timestamp вместо длительности.

```php
use App\Cache\Cache;

function summary(Cache $cache, int $userId, callable $load): mixed
{
    return $cache->remember(
        'summary_'.$userId,
        new \DateInterval('PT5M'),
        $load,
    );
}
```

`remember(string $key, int|DateInterval|null $ttl, callable $callback): mixed` вызывает callback только при промахе.
`rememberForever(string $key, callable $callback): mixed` использует TTL `null`.
В кэше можно хранить даже `null`: наличие значения проверяется через `isHit()`, а не сравнением с `null`.

## Инвалидация и ограничения

Проверить ключ можно через `has(string $key): bool`, удалить - через `delete(string $key): bool`.
После изменения исходных данных удаляйте относящиеся к ним ключи или выбирайте достаточно короткий TTL.
Для персонального результата включайте в ключ идентификатор пользователя, чтобы чужие значения не смешивались.

В этой обертке нет очистки всего пула, тегов, атомарных счетчиков и блокировки вычисления.
Два параллельных промаха `remember()` могут одновременно выполнить callback.
Для платежа или другого внешнего действия нужна отдельная защита: одного этого метода недостаточно.
Есть еще нюанс: `remember()` не проверяет результат `save()` и вернет вычисленное значение, даже если сохранить его не удалось.
Для явной записи результат `set()` доступен вызывающему коду.
