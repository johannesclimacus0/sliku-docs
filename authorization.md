# Авторизация

Аутентификация помогает узнать пользователя, а авторизация - проверить, что ему разрешено делать.
Проверки прав собраны в `App\Security\Authorization` и Symfony Voters.

## Проверка действия

`Authorization::isGranted(User $user, string $action, mixed $subject = null)` возвращает bool.
Если разрешения нет, `denyAccessUnlessGranted()` с теми же аргументами выбросит `App\Exceptions\ForbiddenException`.
Если пользователь еще не сохранен и у него нет идентификатора, проверка вернет отказ.

Например, перед изменением профиля можно проверить, вправе ли текущий пользователь редактировать выбранную учётную запись:

```php
use App\Modules\Account\Security\UserVoter;
use App\Security\Authorization;

final class ProfileEditor
{
    public function __construct(
        private Authorization $authorization,
    ) {}

    public function update(User $actor, User $target, string $name, ?string $bio): void
    {
        $this->authorization->denyAccessUnlessGranted(
            $actor,
            UserVoter::EDIT,
            $target,
        );

        $target->updateProfile($name, $bio);
    }
}
```

Здесь `$actor` - пользователь, выполняющий действие, а `$target` - профиль, который он хочет изменить.

В HTTP-контроллере текущего пользователя можно получить через `AuthenticatedUser::fromRequest($request)`, если запрос уже прошёл `JwtAuthMiddleware`. 

## Правила пользователей

В модуле `Account` правила собраны в `App\Modules\Account\Security\UserVoter`. 
Он поддерживает просмотр пользователя, изменение профиля и управление пользователями.

Для просмотра и редактирования в качестве `subject` передаётся объект `App\Entity\User`. 
Для управления пользователями используется `User::class`:

```php
use App\Entity\User;
use App\Modules\Account\Security\UserVoter;

$authorization->denyAccessUnlessGranted(
    $actor,
    UserVoter::MANAGE,
    User::class,
);
```

Администратор может выполнять все поддерживаемые действия. Обычный пользователь может просматривать и изменять свой профиль, но не управлять другими пользователями. Неизвестные действия не получают разрешение автоматически.

Важно передавать `subject` в том виде, который ожидает voter. Например, при проверке прав на конкретного пользователя нужен объект `User`, а не его числовой идентификатор.

## Роли

В стартере предусмотрены две роли на уровне приложения: `member` и `admin`. Они определены в `UserRole`. Метод `User::getRoles()` возвращает `ROLE_USER` и дополнительную роль пользователя: `ROLE_MEMBER` или `ROLE_ADMIN`.

Вместе с `UserVoter` подключён Symfony `RoleVoter`. Поэтому можно выполнять и проверки по имени роли:

```php
$authorization->isGranted($actor, 'ROLE_ADMIN');
```

Отдельная иерархия ролей не настроена. Наличие `ROLE_ADMIN` само по себе не означает, что Symfony автоматически выдаёт все остальные роли. Доступ к конкретным действиям определяется правилами соответствующих voters.

Для локальной разработки роль пользователя можно изменить консольной командой:

```bash
./run console user:role member@example.test admin
```

## Добавление собственных правил

Список voters находится в `config/registrations/voters.php`:

```php
use App\Modules\Account\Security\UserVoter;
use Symfony\Component\Security\Core\Authorization\Voter\RoleVoter;

return [UserVoter::class, RoleVoter::class];
```

Для нового правила создаётся класс на основе Symfony `Voter`, в котором `supports()` определяет поддерживаемые действия и типы объектов, а `voteOnAttribute()` решает, разрешено ли действие конкретному пользователю.

После этого класс нужно добавить в список регистраций. В `config/definitions/security.php` контейнер создаёт зарегистрированные voters и передаёт их в `AccessDecisionManager`. 
Само размещение класса в каталоге `Security/` ещё не подключает его к проверкам.

Если одно действие поддерживают несколько voters, окончательное решение зависит от стратегии `AccessDecisionManager`. 
Поэтому полезно заранее определить, за какие действия отвечает каждое правило.