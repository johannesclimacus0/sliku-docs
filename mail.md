# Почта

Для отправки электронных писем в стартере используется `App\Mail\MailClient` на основе Symfony Mailer. 
Он позволяет отправлять обычные текстовые сообщения, HTML-письма и письма с Twig-шаблонами.

По умолчанию `MailClient` отправляет сообщение сразу при вызове метода. 
Если отправка не должна задерживать HTTP-ответ, её можно перенести в отдельное задание [очереди](/docs/{{version}}/queues).

## Настройка отправки

Параметры почты находятся в `config/settings/mail.php` и задаются через `.env`:

```dotenv
MAILER_DSN=smtp://mailpit:1025
MAIL_FROM_ADDRESS=no-reply@example.test
MAIL_FROM_NAME=App
```

Для локальной разработки в Docker уже доступен Mailpit, где можно просматривать отправленные письма без доставки настоящим адресатам. 
Для реальной отправки потребуется настроить `MAILER_DSN` вашего SMTP-сервера.

Сервис создаётся в `config/definitions/mail.php`.
Если `MAILER_DSN` не указан, приложение выдаст `LogicException` с сообщением `MAILER_DSN must be configured` при создании соответствующей зависимости.

Метод `MailClient::send(Email $email): void` работает с копией переданного письма. 
Если у него нет отправителя `From`, используются `mail.from_address` и `mail.from_name`. 
Затем при необходимости обрабатывается шаблон, а готовое письмо передаётся Symfony Mailer.

## Отправка простого письма

`MailClient` можно получить через DI, указав его в конструкторе сервиса:

```php
use App\Mail\MailClient;

final class ReportMailer
{
    public function __construct(
        private MailClient $mail,
    ) {}

    public function ready(string $email): void
    {
        $this->mail->text(
            $email,
            'Отчёт готов',
            'Отчёт доступен в личном кабинете.',
        );
    }
}
```

Метод `text(string $to, string $subject, string $body): void` подходит для обычных текстовых сообщений. 
Если нужны HTML-содержимое, вложения или дополнительные заголовки, можно создать объект `Symfony\Component\Mime\Email` и передать его в `send()`.

## Шаблоны Twig

Для писем с динамическим содержимым можно использовать Twig. Шаблоны хранятся в `templates/`, а соответствующие зависимости настроены в `config/definitions/twig.php`.

Например, письмо с уведомлением о готовом отчёте:

```php
use App\Mail\MailClient;
use Symfony\Bridge\Twig\Mime\TemplatedEmail;

function sendReport(MailClient $mail, string $to, string $name): void
{
    $email = (new TemplatedEmail())
        ->to($to)
        ->subject('Отчёт готов')
        ->htmlTemplate('emails/report-ready.html.twig')
        ->context(['name' => $name]);

    $mail->send($email);
}
```

Шаблон `templates/emails/report-ready.html.twig`:

```twig
<p>Здравствуйте, {{ name }}!</p>
<p>Ваш отчёт готов.</p>
```

В Twig включён режим `strict_variables`, поэтому обращение к переменной, которой нет в контексте, приведёт к ошибке. 
В общий контекст рендерера также добавлено значение `appName`.

Пример подготовки письма в существующем модуле можно посмотреть в `src/Modules/Account/Mail/VerificationReminder.php`: метод `create(User $user): TemplatedEmail` формирует письмо с данными пользователя и необходимой ссылкой.

## Отправка через очередь

Для фоновой отправки письмо формируется и отправляется внутри обработчика.
В само сообщение обычно достаточно передать идентификатор пользователя или другие необходимые данные, а актуальную информацию получить уже при выполнении задания.

Сообщение и его обработчик регистрируются в `config/registrations/jobs.php`. 

Для его запуска worker в Docker используется команда:

```bash
./run --profile queue up -d worker
```

При прямом вызове `text()` или `send()` отдельный worker не нужен.