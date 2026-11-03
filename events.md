# События

События позволяют разным частям приложения реагировать на произошедшие изменения, не связывая их напрямую. 

В стартере используется Symfony EventDispatcher. 
Его настройки находятся в `config/definitions/events.php`, а события и слушатели приложения - в соответствующих каталогах модулей.

## Создание и отправка события

Событие представляет собой обычный PHP-объект с данными о произошедшем действии. 

```php
use App\Entity\User;
use App\Modules\Account\Events\UserProfileUpdated;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

final class ProfileEvents
{
    public function __construct(
        private EventDispatcherInterface $events,
    ) {}

    public function changed(User $user): void
    {
        $this->events->dispatch(new UserProfileUpdated($user, ['name']));
    }
}
```

`EventDispatcherInterface` передаётся через DI-контейнер. При вызове `dispatch()` диспетчер находит зарегистрированных слушателей по классу события и вызывает их.

Само событие не сохраняет изменения в базе данных. 
Его стоит отправлять в подходящий момент сценария, учитывая, что слушатели могут выполнять дополнительные действия. 
Для создания `UserEvent` пользователь уже должен иметь идентификатор, иначе конструктор выбросит исключение.

Константа `NAME` не используется для поиска слушателей по классу. 
Она нужна там, где событие должно иметь понятное строковое обозначение, например в уведомлениях или вебхуках.

## Слушатели событий

Слушатель выполняет действие в ответ на определённое событие.

Например, слушатель изменения профиля:

```php
namespace App\Modules\Account\Listeners;

use App\Modules\Account\Events\UserProfileUpdated;
use Psr\Log\LoggerInterface;

final class ProfileAuditListener
{
    public function __construct(
        private LoggerInterface $logger,
    ) {}

    public function __invoke(UserProfileUpdated $event): void
    {
        $this->logger->info('Profile updated', [
            'user_id' => $event->userId,
        ]);
    }
}
```

Слушатели регистрируются в `config/registrations/events.php`. Для каждого класса события можно указать один или несколько обработчиков:

```php
use App\Modules\Account\Events\UserProfileUpdated;
use App\Modules\Account\Listeners\ProfileAuditListener;

return [
    'subscribers' => [
        // Подписчики приложения.
    ],
    'listeners' => [
        UserProfileUpdated::class => [
            ProfileAuditListener::class,
        ],
    ],
];
```

Контейнер создаёт слушателей и внедряет их зависимости. 
Но само наличие класса в `Listeners/` не подключает его к диспетчеру. Даже после создания заготовки через `make:listener` нужна регистрация.

## Подписчики

Если один класс должен реагировать на несколько событий, можно использовать подписчик с интерфейсом `EventSubscriberInterface`.

Подписчик объявляет статический метод `getSubscribedEvents(): array`, в котором сопоставляет события и методы обработки. 
После этого его класс добавляется в массив `subscribers` в `config/registrations/events.php`.

## Синхронное выполнение и очереди

По умолчанию `dispatch()` вызывает слушателей синхронно, в рамках текущего процесса. 
Это означает, что выполнение запроса продолжается только после завершения слушателей, а исключение внутри слушателя может прервать текущую операцию.

Для длительных действий слушатель может поставить отдельное задание в очередь через `Queue::dispatch()` или сервис, который использует очередь. 
Тогда обработку задания выполнит worker.

События можно отправлять без запущенного worker, но задания, помещённые в очередь, не будут обработаны, пока worker не начнёт их выполнять. 
Подробнее это разобрано в разделе [Очереди](/docs/{{version}}/queues).

Также важно учитывать транзакции базы данных.
В текущей конфигурации диспетчер не откладывает события автоматически до `commit`. 
Если событие отправлено внутри транзакции, его слушатели могут выполниться ещё до сохранения изменений. 