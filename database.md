# База данных

Для хранения данных стартер использует PostgreSQL. 
Основная работа с сущностями построена на Doctrine ORM, а для служебных таблиц и запросов, где удобнее работать напрямую с SQL, используется Doctrine DBAL.

Сущности описаны PHP-атрибутами в `src/Entity/`. 

## Подключение

Параметры подключения задаются в `.env` и собираются в `config/settings/database.php`:

- `DB_HOST` - адрес сервера PostgreSQL.
- `DB_PORT` - порт подключения.
- `DB_NAME` - имя базы данных.
- `DB_USER` и `DB_PASSWORD` - учётные данные.
- `DB_CONNECT_TIMEOUT` - время ожидания подключения.

По умолчанию используется драйвер `pdo_pgsql`, хост `db` и таймаут в три секунды. 
В Docker Compose запускается PostgreSQL 17. 
Данные сохраняются в томе `postgres_data`, а порт `5432` доступен на loopback-интерфейсе хоста.

Doctrine настраивается в `config/definitions/doctrine.php`. 
Здесь создаётся `EntityManager`, который через DI-контейнер доступен по интерфейсу `EntityManagerInterface`. 
Для метаданных сущностей из `src/Entity/` используется кэш PSR-6, а для отложенной загрузки - native lazy objects.
Если включён Clockwork, к DBAL подключается профилирование SQL-запросов.

## Работа с сущностями

Для работы с Doctrine достаточно указать `EntityManagerInterface` в конструкторе сервиса или репозитория. Контейнер передаст настроенный менеджер автоматически.

Ниже упрощённый пример сохранения сущности:

```php
use App\Entity\User;
use Doctrine\ORM\EntityManagerInterface;

final class RegisterMember
{
    public function __construct(
        private EntityManagerInterface $em,
    ) {}

    public function create(string $email, string $password): User
    {
        $user = new User($email, password_hash($password, PASSWORD_DEFAULT));

        $this->em->persist($user);
        $this->em->flush();

        return $user;
    }
}
```

`persist()` сообщает Doctrine, что сущность нужно отслеживать для сохранения, а `flush()` выполняет накопленные изменения в базе данных.

Если несколько операций должны выполняться как единое целое, используется `wrapInTransaction()`. 

Для прямых SQL-запросов можно получить соединение через `$em->getConnection()` и передавать значения параметрами, не собирая SQL из пользовательского ввода.

## Миграции

```sh
./run doctrine migrations:diff
./run migrate --no-interaction
./run console db:seed
```

Миграции лежат в `database/migrations`, служебная таблица версий - `doctrine_migration_versions`.
Настройки находятся в `database/migration.php`.

### Таблицы вне ORM

Часть служебных таблиц не описана сущностями Doctrine. Они перечислены в настройке `database.unmanaged_tables`:

- `refresh_sessions`
- `messenger_messages`
- `webhook_deliveries`
- `webhook_attempts`
- `user_activity`
- `notifications`

Эти таблицы исключены из сравнения схемы ORM. 
Поэтому `migrations:diff` не создаёт для них изменения по PHP-сущностям, а необходимые SQL-миграции пишутся вручную.

Перед применением новой миграции стоит просмотреть сгенерированный SQL, особенно если в нём есть удаление таблиц или столбцов.
