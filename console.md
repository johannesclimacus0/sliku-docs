# Консоль

В стартере используется Symfony Console. 
Через консоль можно выполнять миграции, управлять очередями, запускать задачи по расписанию, работать с пользователями и создавать заготовки новых классов.

Точка входа находится в `bin/console`. 
За создание консольного приложения отвечает `app/Console/ConsoleFactory.php`: фабрика загружает `.env`, получает DI-контейнер и собирает Slim Console с ленивой загрузкой команд.

## Запуск команд

В Docker удобнее использовать скрипт `run`, который выполняет команды в окружении приложения:

```bash
./run console list
./run console help users:remind
./run console queue:stats
```

При локальной установке PHP и зависимостей команды можно запускать напрямую:

```bash
php bin/console list
```

Для просмотра доступных аргументов и опций используется `help`. Особенно полезно проверять справку перед командами, которые изменяют или удаляют данные.

Консоль использует тот же DI-контейнер, что и HTTP-приложение. 
Поэтому в команды можно передавать через конструктор сервисы, репозитории, настройки и логгер без ручного создания зависимостей внутри `execute()`.

## Собственная команда

Команда представляет собой PHP-класс на основе Symfony `Command`. 
Например, файл `src/Modules/Account/Commands/GreetCommand.php` может выглядеть так:

```php
namespace App\Modules\Account\Commands;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

final class GreetCommand extends Command
{
    public function __construct()
    {
        parent::__construct('app:greet');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln('Приложение готово');

        return Command::SUCCESS;
    }
}
```

Новый класс необходимо зарегистрировать в `config/registrations/commands.php`, сохранив существующие записи:

```php
'app:greet' => \App\Modules\Account\Commands\GreetCommand::class,
```

После этого команда станет доступна через:

```bash
./run console app:greet
```

Аргументы и опции добавляются в `configure()` с помощью `InputArgument` и `InputOption`. 

Если команда не смогла выполнить задачу, она должна возвращать `Command::FAILURE`. Так ошибка будет видна вызывающему процессу, например cron.

## Генераторы классов

Чтобы не создавать однотипные файлы вручную, в стартере предусмотрены генераторы. 
Поддерживаются контроллеры, сервисы, команды, задания, обработчики, события, слушатели, voters, DTO, репозитории и middleware.

```bash
./run console make:command Greet --module=Account
./run console make:service Reports/Export --module=Account
./run console make:job Export --module=Account
```

Список типов задаётся в `ClassGenerator::TYPES` (`app/Console/ClassGenerator.php`). 
Генератор размещает файлы в `src/Modules/`, создаёт соответствующий namespace и добавляет суффикс по типу класса.

Например, `make:service Reports/Export --module=Account` создаст `src/Modules/Account/Services/Reports/ExportService.php`.

Имя задаётся в UpperCamelCase, для вложенных каталогов можно использовать `/`. 
Параметр `--module` по умолчанию равен `Account`. Существующие файлы генератор не перезаписывает.

Генератор создаёт только заготовку класса. 
Маршруты, команды, слушатели, обработчики очередей и voters после этого нужно зарегистрировать в соответствующих файлах конфигурации.

## Терминальный интерфейс

Для управления проектом без постоянного ввода команд доступна [терминальная панель](/docs/{{version}}/tui):

```bash
./run ui
```

Через неё можно просматривать состояние контейнеров, очереди, расписание, вебхуки и логи, а также запускать генераторы и команды в разных окружениях.

Оформление терминальных рисунков настраивается в отдельном [редакторе артов](/docs/{{version}}/tui#редактор-артов):

```bash
./run art:editor
```

## Служебные команды

Для повседневной разработки и обслуживания приложения предусмотрены отдельные группы команд:

| Область      | Команды                                                                    |
|--------------|----------------------------------------------------------------------------|
| Очереди      | `queue:work`, `queue:stats`, `queue:failed`, `queue:retry`, `queue:forget` |
| Планировщик  | `schedule:run`                                                             |
| Пользователи | `db:seed`, `user:role`, `users:remind`                                     |
| Сессии       | `sessions:prune`                                                           |

Подробнее об их использовании рассказано в разделах [Очереди](/docs/{{version}}/queues) и [Планировщик](/docs/{{version}}/scheduling). 
Полный список команд и точные параметры доступны через `list` и `help`.