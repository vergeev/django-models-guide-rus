# Взаимодействие фреймворка Django с различными СУБД на примере простой системы управления персоналом
## Подготовительная часть
### Автономное окружение
В этом руководстве мы будем использовать Python версии 3 и Django версии 1.11. Обе версии являются актуальными на время написания этого руководства.

Чтобы не привязываться к конкретной IDE, работать с Python мы будем в командной строке.
Ее можно открыть, нажав “лупу” в левом нижнем углу экрана и набрав `cmd.exe`.

На компьютерах РЭУ им. Г.В. Плеханова Python установлен в папке Python36-32 на диске C. Чтобы перейти в эту папку в командной строке, наберем `cd "C:\Program Files\Python36-32"`.

Чтобы посмотреть, какие файлы расположены в папке, наберем `dir`. В появившемся списке должен быть `python.exe`.

Мы собираемся изолировать наше будущее приложение от глобального окружения. Это позволит не перемещаться всякий раз на диск С для того, чтобы использовать Python. Для этого создадим виртуальное окружение:
1. Создадим на рабочем столе папку `django_hrm`: `mkdir C:\Users\USER\Desktop\django_hrm`
2. И разместим в ней виртуальное окружение: `python -m venv C:\Users\USER\Desktop\django_hrm\venv`
3. Теперь активируем его: `C:\Users\USER\Desktop\django_hrm\venv\Scripts\activate.bat`
4. В начале строки появится приписка (venv), которая показывает, что виртуальное окружение активно. Наконец, чтобы деактивировать виртуальное окружение, достаточно написать `deactivate`.

Теперь в папке `django_hrm` будет располагаться всё необходимое для запуска проекта, так что ее можно переносить с компьютера на компьютер. Ее также можно расположить где угодно и назвать как угодно, но это руководство будет дальше предполагать, что она расположена на рабочем столе и называется именно так.

Чтобы убедиться, что все подготовлено правильно, запустим тестовое приложение.
1. Переместимся в папку django_hrm: `cd C:\Users\USER\Desktop\django_hrm`
2. Активируем виртуальное окружение: `venv/Scripts/activate.bat`
3. Установим веб-фреймворк Django, с которым и будем работать в дальнейшем. Для этого воспользуемся встроенным в Python пакетным менеджером pip: `pip install django==1.11.0`
4. Создадим наш проект: `django-admin startproject django_hrm .` 

    Разберем эту команду подробнее. Команда `django-admin` появилась в виртуальном окружении после установки Django. `django_hrm` — называние нашего проекта, а точка указывает, что файлы проекта нужно поместить в текущую директорию.
5. Напишем `dir` и убедимся, что появился файл `manage.py` и папка `django_hrm`. Об их содержимом мы поговорим позже, а пока просто запустим наш сайт: `python manage.py runserver`
6. Откроем в браузере http://127.0.0.1:8000/. На странице должно говориться "It worked!"

### Структура Django-приложения
Итак, помимо виртуального окружения и папки нашего проекта, у нас появился файл `manage.py`. Он используется для управления проектом: запуска тестового сервера (см. выше), создания модулей, проведения миграций базы данных. Мы еще не раз будем использовать этот файл.

В папке `django_hrm` находятся файлы `wsgi.py`, `urls.py` и `settings.py`. Первый нужен для запуска проекта на серверах вроде nginx и Apache (эта тема здесь рассматриваться не будет). `urls.py` используется для того, чтобы указать фреймворку, какой код какому URL соответсвует. Наконец, `settings.py` содержит настройки проекта, в том числе интересующие нас настройки подключения к базе данных.

> Прямо «из коробки», Django использует легковесную СУБД SQLite3 и сохраняет ее в файле `db.sqlite3`, который сохраняется в папке с `manage.py`. SQLite3 подходит для отладки и тестирования несложных проектов, но при запуске ее обычно меняют на MySQL или Postgres.

Проект состоит из модулей, или _приложений_. У нас будет лишь одно приложение, которое мы назовем `core`. Создадим его следующим образом:
1. Переместимся в папку `django_hrm` и активируем виртуальное окружение (см. выше инструкции по тому, как это сделать).
2. Создадим новое приложение командой `python manage.py startapp core`.

В папке с приложением появятся следующие файлы: `admin.py`, `apps.py`, `models.py`, `tests.py`, `views.py`. Далее мы будем работать с `models.py` и `admin.py` (про остальные можно почитать в [документации](https://docs.djangoproject.com/en/1.11/)).

## Проектирование БД
### Django ORM
Согласно Википедии, ORM (Object-Relational Mapping, рус. объектно-реляционное отображение) — это технология программирования, которая связывает базы данных с концепциями объектно-ориентированных языков программирования, создавая «виртуальную объектную базу данных». Более простым языком — это механизм, который позволит нам описывать таблицы базы данных с помощью объектов.

Django поставляется со своей ORM, которая так и называется, Django ORM. Надо при этом понимать, что ORM бывают разные, и даже Django поддерживает сторонние.

Описывать эти самые классы, которые потом перейдут в таблицы, принято в файле `models.py`. Чтобы его открыть, кликните правой кнопкой мыши и выберите "Edit with IDLE".

### Создание модели и добавление основных полей
Центральным в нашей небольшой системе управления персоналом будет класс сотрудника — `Employee`. Добавим его в `models.py`:

```python
from django.db import models


class Employee(models.Model):
    GENDER_CHOICES = [
        ('M', 'Male'),
        ('F', 'Female'),
        ('O', 'Other'),
    ]

    first_name = models.CharField(max_length=32)
    last_name = models.CharField(max_length=32)
    middle_name = models.CharField(max_length=32, blank=True)
    title = models.CharField(max_length=128)
    gender = models.CharField(max_length=1, choices=GENDER_CHOICES)
    birthday = models.DateField()
    birth_place = models.CharField(max_length=128)
    passport_number = models.CharField(max_length=11, unique=True)
    inn = models.CharField(max_length=12, blank=True)
    snils = models.CharField(max_length=11, blank=True)
```

Итак, на первой строчке мы импортируем модуль `models`, который содержит класс `Model`, от которого наследуется наш класс `Employee`. Это позволяет показать фреймворку, что этот класс мы хотим сделать таблицей.

`first_name`, `last_name` и т.д. — это атрибуты класса, которым присвоены конкретные поля. Поля бывают разные: здесь мы используем `CharField` и `DateField`. В самой БД они конвертируются в `varchar` и `date`. О полях других типов можно прочитать в [документации](https://docs.djangoproject.com/en/1.11/ref/models/fields/). Можно даже добавлять [собственные](https://docs.djangoproject.com/en/1.11/howto/custom-model-fields/) поля.

Конструктор класса `CharField` принимает обязательный атрибут `max_length`, который, как следует из названия, ограничивает максимальную длину строки. При этом можно указать, какие строки допустимы в `CharField`. Это делается аргументом `choices` (см. поле `gender`).

Если мы не указываем первичный ключ сами, Django его "подсовывает" нам автоматически. То есть если у нас нет поля с аргументом `primary_key=True`, Django автоматически добавит следующее:
```python
id = models.AutoField(primary_key=True)
```
`AutoField` — это то же самое, что и `IntegerField`, но с добавлением новых объектов СУБД сама инкрементирует его.

По умолчанию, все поля обязательны для заполнения, то есть создаются с ограничением `NOT NULL`. В Django необходимо двумя способами разрешать указание пустых значений. Во-первых, надо указать другим частям программы (например, формам), что передача пустых значений разрешена, передав аргумент `blank=True`. Во-вторых, указать базе данных, что сохранять пустые значения можно, с помощью `null=True`. Конкретно в случае с `CharField`, по конвенции, передается только аргумент `blank=True`.

> Действительно, только у `CharField` есть два кандидата на пустое значение: это пустая строка и `NULL`. Если указать и `blank=True`, и `null=True`, то может получиться, что в базе окажутся и пустые строки, и `NULL`. По смыслу это одно и то же, а значения разные.
> На самом деле, `ImageField` и `FileField` тоже сохраняются как `varchar` (поскольку сохраняется путь до файла), поэтому их пустые значения тоже принято указывать только с `blank=True`.

Атрибут `unique=True` указывает, что значение должно быть уникальным.

> На самом деле, поля `inn` и `snils` должны быть уникальными. Однако из-за аргумента `blank=True` они могут сохраняться как пустые строки, а пустые строки между собой равны. Это не позволит создать нескольких людей без ИНН и СНИЛС. Поэтому уникальность этих полей придется проверять не на уровне базы данных, а при валидации пользовательского ввода, что выходит за рамки этой статьи.

Покажем, как пользоваться нашей моделью.
1. Переместимся в папку с проектом и активируем виртуальное окружение.
2. В файле `settings.py` найдем список `INSTALLED_APPS` и добавим в его начало строку `'apps.CoreConfig',`. В итоге он должен выглядеть следующим образом:
    ```
    INSTALLED_APPS = [
        'core.apps.CoreConfig',
        'django.contrib.admin',
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.messages',
        'django.contrib.staticfiles',
    ]
    ```
    Это необходимо, чтобы следующие команды нашли наши модели.
3. Запустим команду `python manage.py makemigrations`. Так Django сгенерирует команды, необходимые для добавления в нашу базу данных таблицы `Employee`.
4. Запустим команду `python manage.py migrate`. Так Django применит сгенерированные команды к базе данных, создав нужные таблицы.
5. Чтобы посмотреть, какой SQL применем к базе данных во время миграций, наберем `python manage.py sqlmigrate core 0001_initial`. Результат этой команды выглядит так:
    ```sql
    BEGIN;
    --
    -- Create model Employee
    --
    CREATE TABLE "core_employee" (
        "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
        "first_name" varchar(32) NOT NULL,
        "last_name" varchar(32) NOT NULL,
        "middle_name" varchar(32) NOT NULL,
        "title" varchar(128) NOT NULL,
        "gender" varchar(1) NOT NULL,
        "birthday" date NOT NULL,
        "birth_place" varchar(128) NOT NULL,
        "passport_number" varchar(11) NOT NULL UNIQUE,
        "inn" varchar(12) NOT NULL,
        "snils" varchar(11) NOT NULL
    );
    COMMIT;
    ```
    Здесь мы видим, как Django обращается с таблицами. Во-первых, нам отобразился искуственный ключ, который Django создает для моделей, для которых первичный ключ не указан. Во-вторых, название таблицы имеет формат `<название приложения>_<название модели строчными буквами>`, что позволяет иметь одинаково названные модели в разных приложениях.

    Это SQL-синтаксис для СУБД SQLite, поскольку наше приложение подключено именно к ней. Для разных СУБД Django может генерировать разный SQL.
6. Наберем `python manage.py shell`. Откроется интерактивная консоль, из которой мы сможем пользоваться нашей моделью.
7. В этой консоли наберем, одну за другой, следующие команды:
    ```python
    >>> from core.models import Employee
    >>> import datetime
    >>> employee = Employee(first_name='Джон', last_name='До', title='Разработчик', gender='M', birthday=datetime.datetime(1984, 1, 1), birth_place='Луна', passport_number='12345678900')
    >>> employee.first_name
    'Джон'
    >>> employee.middle_name == ''
    True
    >>> employee.middle_name is None
    False
    >>> employee.id is None
    True
    >>> employee.save()
    >>> employee.id is None
    False
    ```
    Заметим следующие вещи:
    - Хотя в SQL выше `middle_name` объявлено как `NOT NULL`, мы сохранили `employee` без ошибок. Все потому, что сохранилось не значение `NULL`, а пустая строка. Это произошло из-за `blank=True`.
    - До того, как мы сохранили объект, его `id` был `NULL`. Из-за ключевого слова `AUTOINCREMENT` СУБД только во время добавления может определить `id` объекта.
8. Любой Django-проект по умолчанию находится в режиме отладки (строчка `DEBUG=True` в `settings.py`). В этом режиме он записывает время и содержание всех SQL-запросов. Набрав следующее в интерактивную консоль, мы можем просмотреть эти записи:
    ```python
    >>> from django.db import connection
    >>> for query in connection.queries:
    ...  print(query['sql'])
    ...
    BEGIN
    INSERT INTO "core_employee" ("first_name", "last_name", "middle_name", "title", "gender", "birthday", "birth_place", "passport_number", "inn", "snils") VALUES ('Джон', 'До', '', 'Разработчик', 'M', '1984-01-01', 'Луна', '12345678900', '', '')
    ```
    Как мы видим, Django сама сгенерировала правильный SQL-запрос, когда мы ее попросили сохранить нашу модель в базу данных.
9. Чтобы закрыть интерактивную консоль, наберите `exit()`.

### Добавление внешних ключей
Наш сотрудник может иметь несколько телефонов и почтовых ящиков. Чтобы добавить такое «один-к-многим» отношение создадим таблицы телефонных номеров и почтовых ящиков, добавив к каждому из них внешний ключ, указывающий на сотрудника-владельца. Для этого добавим в конец `models.py`:
```python
class Phone(models.Model):
    number = models.CharField(max_length=12)
    owner = models.ForeignKey(Employee, on_delete=models.CASCADE)


class Email(models.Model):
    email = models.EmailField()
    owner = models.ForeignKey(Employee, on_delete=models.CASCADE, related_name='emails')
```
Аргумент `on_delete` указывает, что должно случиться с объектом, если соответствующий ему `Employee` удаляется. Возможные значения (`CASCADE`, `PROTECT`, `SET_NULL`, `SET_DEFAULT`, `DO_NOTHING`) имеют интуитивно-понятные названия и в подробностях описаны в [документации](https://docs.djangoproject.com/en/1.11/ref/models/fields/#django.db.models.ForeignKey.on_delete).

Посмотрим, как эти модели представлены в SQL и как ими пользоваться:
1. `python manage.py makemigrations`
2. `python manage.py migrate`
3. Команда `python manage.py sqlmigrate core 0002_email_phone` выдаст следующее:
    ```sql
    BEGIN;
    --
    -- Create model Email
    --
    CREATE TABLE "core_email" (
        "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
        "email" varchar(254) NOT NULL,
        "owner_id" integer NOT NULL REFERENCES "core_employee" ("id")
    );
    --
    -- Create model Phone
    --
    CREATE TABLE "core_phone" (
        "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
        "number" varchar(12) NOT NULL,
        "owner_id" integer NOT NULL REFERENCES "core_employee" ("id")
    );
    CREATE INDEX "core_email_owner_id_fceb92d8" ON "core_email" ("owner_id");
    CREATE INDEX "core_phone_owner_id_48bc00df" ON "core_phone" ("owner_id");
    COMMIT;
    ```
    Здесь мы видим, что полю `EmailField` соответствует тип `varchar(254)`. Дейтсвительно, на уровне базы данных почтовый ящик достаточно хранить в качествае строки. А вот уже на уровне приложения имеет смысл проверять, действительно ли полученная строка является электронной почтой. Именно для этой проверки в Django и существует `EmailField`.
4. `python manage.py shell`
5. Сохраним почтовый ящик и номер телефона для сотрудника, созданного ранее:
    ```python
    >>> from core.models import Email
    >>> email = Email(email='legit@email.ru', owner_id=1)
    >>> email.save()
    >>> from core.models import Phone
    >>> phone = Phone(number='+79991234567', owner_id=1)
    >>> phone.save()
    ```
6. Возьмем ранее созданного сотрудника (более детально написание запросов мы рассмотрим далее):
    ```python
    >>> from core.models import Employee
    >>> employee = Employee.objects.first()
    ```
7. Получим его номер телефона и почтовый ящик:
    ```python
    >>> employee.emails.first().email
    'legit@email.ru'
    >>> employee.phone_set.first().number
    '+79991234567'
    ```
    Заметим, что к почтовым ящикам мы обратились как `emails`, а к телефонам `phone_set`. Название `emails` мы установили выше в аргументе `related_name`. Когда этот аргумент не указан, `related_name` генерируется автоматически и выглядит как `<название модели строчными буквами>_set`.
8. Фактически выполненные SQL-запросы мы можем посмотреть следующим образом:
    ```python
    >>> from django.db import connection
    >>> print(connection.queries[-1]['sql'])
    SELECT "core_phone"."id", "core_phone"."number", "core_phone"."owner_id" FROM "core_phone" WHERE "core_phone"."owner_id" = 1 ORDER BY "core_phone"."id" ASC LIMIT 1
    >>> print(connection.queries[-2]['sql'])
    SELECT "core_email"."id", "core_email"."email", "core_email"."owner_id" FROM "core_email" WHERE "core_email"."owner_id" = 1 ORDER BY "core_email"."id" ASC LIMIT 1
    ```
Итак, мы рассмотрели, как реализуется отношение «один-к-многим».

### Добавление отношений «многие-к-многим»
Сотрудник может владеть несколькими языками, и несколько сотрудников могут владеть одним языком. Это пример отношения «многие-к-многим». Чтобы его реализовать моделями в Django, добавим следующий код в конец файла `models.py`:
```python
class Language(models.Model):
    name = models.CharField(max_length=20)
    employees = models.ManyToManyField(Employee)
```
Сгенерировав миграции (см. команды для этого выше) и выполнив `python manage.py sqlmigrate core 0003_language`, увидим следующее:
```sql
BEGIN;
--
-- Create model Language
--
CREATE TABLE "core_language" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "name" varchar(20) NOT NULL
);
CREATE TABLE "core_language_employees" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "language_id" integer NOT NULL REFERENCES "core_language" ("id"),
    "employee_id" integer NOT NULL REFERENCES "core_employee" ("id")
);
CREATE UNIQUE INDEX "core_language_employees_language_id_employee_id_138ab01e_uniq" ON "core_language_employees" ("language_id", "employee_id");
CREATE INDEX "core_language_employees_language_id_828dc2f8" ON "core_language_employees" ("language_id");
CREATE INDEX "core_language_employees_employee_id_b547f345" ON "core_language_employees" ("employee_id");
COMMIT;
```
Как мы видим из этого листинга, Django сама создает промежуточную таблицу для отношения.

Если мы захотим сохранить дополнительную информацию вместе с нашим отношением, нам придется сделать таблицу самостоятельно. Перепишем класс `Language` следующим образом:
```python
class Language(models.Model):
    name = models.CharField(max_length=20)
    employees = models.ManyToManyField(
        Employee,
        related_name='languages',
        through='LanguageEmployee'
    )


class LanguageEmployee(models.Model):
    SKILL_LEVELS = [
        ('E', 'Elementary'),
        ('I', 'Intermediate'),
        ('A', 'Advanced'),
    ]
    skill = models.CharField(max_length=1, choices=SKILL_LEVELS)
    employee = models.ForeignKey(Employee, on_delete=models.DO_NOTHING)
    language = models.ForeignKey(Language, on_delete=models.DO_NOTHING)
```
Как только мы сгенерируем и применим миграции, мы получим ошибку. Это иллюстрирует один из главных недостатков ORM: иногда преобразования настолько сложны, что она не в состоянии сгенерировать валидный SQL.

Если бы у нас были пользовательские данные, то мы бы отредактировали миграции [таким образом](https://stackoverflow.com/a/40654521/3694363), чтобы их не потерять эти данные. Но поскольку модель `Language` мы только-только создали, нам достаточно отменить последнюю миграцию: `python manage.py migrate core 0002_email_phone`. За этим в папке `core/migrations/` удалим файл `0003_language.py` и файл с названием, начинающимся на `0004_` (две последние миграции).

Теперь знакомые нам команды `python manage.py makemigrations` и `python manage.py migrate` выполнятся успешно и создадут нужные таблицы.

Наконец, разберемся, как этим пользоваться. Выполнив `python manage.py shell`, наберем следующее:
```python
>>> from core import models
>>> english = models.Language(name='English')
>>> english.save()
>>> relationship = models.LanguageEmployee(language=english, employee=employee, skill='A')
>>> relationship.save()
>>> employee = models.Employee.objects.first()
>>> employee.languages.first().name
'English'
>>> english.employees.first().first_name
'Джон'
```
Теперь мы видим, что отношение работает. Заинтересованному читателю предлагаем посмотреть, какой SQL генерирует ORM при этих запросах.

### Наследование моделей
В нашей организации может быть не один вид сотрудника. Как минимум бывают постоянные сотрудники ("employees") и контрактники ("contractors"). У них есть как общие, так и различные поля, которые нам хотелось бы отразить в нашей модели данных.

Итак, заменим наш класс `Employee` на три:
```python
class Worker(models.Model):
    GENDER_CHOICES = [
        ('M', 'Male'),
        ('F', 'Female'),
        ('O', 'Other'),
    ]

    first_name = models.CharField(max_length=32)
    last_name = models.CharField(max_length=32)
    middle_name = models.CharField(max_length=32, blank=True)
    title = models.CharField(max_length=128)
    gender = models.CharField(max_length=1, choices=GENDER_CHOICES)
    birthday = models.DateField()
    birth_place = models.CharField(max_length=128)
    passport_number = models.CharField(max_length=11, unique=True)
    inn = models.CharField(max_length=12, blank=True)
    snils = models.CharField(max_length=11, blank=True)

    class Meta:
        abstract = True


class Employee(Worker):
    employment_date = models.DateField()


class Contractor(Worker):
    contract_number = models.CharField(max_length=15, unique=True)
```
Класс `Meta` содержит информацию, необходимо фреймворку для работы с моделью. В частности, строка `abstract = True` означает, что для модели не надо создавать таблицу в базе данных (опять же, заинтересованному читателю рекомендуем посмотреть SQL-запрос, генерируемый миграцией). 

Так как мы добавили поле `employment_date` к модели `Employee`, Django нас попросит предоставить значение для уже созданного объекта или предоставить значение по умолчанию. Выберем первую опцию и введем значение `datetime.date.today()`, то есть скажем, что уже созданный сотрудник у нас был принят на работу сегодня:
```
You are trying to add a non-nullable field 'employment_date' to employee without a default; we can't do that (the database needs something to populate existing rows).
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows with a null value for this column)
 2) Quit, and let me add a default in models.py
Select an option: 1
Please enter the default value now, as valid Python
The datetime and django.utils.timezone modules are available, so you can do e.g. timezone.now
Type 'exit' to exit this prompt
>>> datetime.date.today()
```
Применим миграции. Теперь откроем интерактивную консоль и создадим контрактного работника:
```python
>>> from core.models import Contractor
>>> import datetime
>>> contractor = Contractor(first_name='Иван', last_name='Иванов', gender='M', birthday=datetime.datetime(1990, 2, 2), birth_place='Марс', passport_number='33353535352', contract_number='123')
>>> contractor.save()
>>> contractor.last_name
'Иванов'
>>> contractor.employment_date
Traceback (most recent call last):
  File "<console>", line 1, in <module>
AttributeError: 'Contractor' object has no attribute 'employment_date'
```
Видим, что у контрактника есть свойства, присущие любому работнику, но не `employment_date`, которое есть только у постоянных работников.

## Написание запросов

### Пара слов о менеджерах
Выше мы видели, что для выполнения некоторых действий (в частности, получение сотрудника, `Employee.objects.first()`), нам понадобился атрибут `objects`. Этот атрибут является объектом класса `Manager`, который отвечает за работу с объектами моделей. Менеджер позволяет создавать объекты, сразу сохраняя их в базе данных (`Employee.objects.create(...)`), удалять из нее (`Employee.objects.delete(...)`), обновлять (`Employee.objects.update(...)`) и, наконец, читать их из базы данных.

Некоторые запросы на чтение из базы данных не оставляют разночтений, и `Manager` возвращает сам объект (таков, например, запрос `Employee.objects.first()`). Другие же запросы могут вернуть несколько значений (`Employee.objects.all()`). Такие запросы возвращают не сами значения, а объект класса `QuerySet`.

Так как типичные реляционные базы данных хранят данные в файловой системе, доступ к ней может быть очень долгим. Как мы увидим дальше, `QuerySet` позволяет избежать лишних запросов к базе данных.

### Запросы `SELECT`
Объекты класса `QuerySet` ленивы. Они не выпустят SQL-запрос до тех пор, пока у них не запросят данные. Поставим эксперимент. В интерактивной консоли (как ее открыть, см. выше) наберем следующее:
```python
>>> from core.models import Employee
>>> queryset = Employee.objects.all()
>>> from django.db import connection
>>> connection.queries
[]
>>> print(queryset)
<QuerySet [<Employee: Employee object>]>
>>> connection.queries
[{'sql': 'SELECT "core_employee"."id", "core_employee"."first_name", "core_employee"."last_name", "core_employee"."middle_name", "core_employee"."title", "core_employee"."gender", "core_employee"."birthday", "core_employee"."birth_place", "core_employee"."passport_number", "core_employee"."inn", "core_employee"."snils", "core_employee"."employment_date" FROM "core_employee" LIMIT 21', 'time': '0.004'}]
```
Мы видим, что до выхова `print(queryset)` к базе данных не осуществлялось никаких запросов.
Это позволяет ставить несколько условий на запрос, не опасаясь лишних походов в файловую систему (пример такого запроса: `Employee.objects.filter(first_name='Женя').exclude(gender='F')`).

Главные операции при работе с объектами класса `QuerySet` — это `filter` и `exclude`, и их названия полностью описывают их применение.

Чтобы задать значения полей более гибко, чем обыкновенным равенством, в SQL используются предикаты вроде `CONTAINS` и `LIKE`. ORM также позволяет подобные вещи. Например, `Employee.objects.filter(first_name__startswith='Е')` эквивалентно `WHERE first_name like "E%"`, а Employee.objects.filter(first_name__contains='Е') — `WHERE first_name contains "E%"`.

### Запросы `CREATE`, `UPDATE` и `DELETE`
Вспомним, как мы создавали объект сотрудника:
```python
>>> from core.models import Employee
>>> import datetime
>>> employee = Employee(first_name='Джон', last_name='До', title='Разработчик', gender='M', birthday=datetime.datetime(1984, 1, 1), birth_place='Луна', passport_number='12345678900')
>>> employee.id is None
True
>>> employee.save()
>>> employee.id is None
False
```
При создании у сотрудника не было первичного ключа `id`, а после вызова метода `save` он появился. Так мы знаем, что Django сделал запрос `CREATE`. Теперь если мы изменим какое-нибудь из свойств объекта `employee` и вызовем `save` еще раз, Django сделает `UPDATE` запрос. Причем этот `UPDATE` будет по всем полям. Если мы изменили, скажем, только имя и фамилию, то сообщить Django об этом можно с помощью аргумента `update_fields`: `employee.save(update_fields=['first_name', 'last_name'])`.

Любопытному читателю рекомендуем проверить, что будет, если мы присвоим `employee.id = None` и вызовем `employee.save()`.

`UPDATE` также можно вызвать и для объекта `QuerySet`, если нужно изменить сразу несоклько объектов: `queryset.update(first_name='Общее имя')`.

Чтобы создать и сохранить в БД обхект в один шаг, используется метод `create`: `new_employee = Employee.objects.create(first_name='Джейн', last_name='До', gender='M', birthday=datetime.datetime(1984, 1, 1), birth_place='Луна', passport_number='12345678900')` 

Удалять объекты можно по одному (`employee.delete()`) или группами (`Employee.objects.delete(gender='M')`).

С помощью Django ORM можно делать и более замысловатые запросы. О них многое написано [в документации](https://docs.djangoproject.com/en/2.0/topics/db/queries/).

### Написание чистых SQL-запросов
В рамках статьи трудно найти задачу, с которой бы не справилась Django ORM. Однако такие существуют: ORM генерирует либо некорректный запрос, либо очень медленный. В таких ситуациях прибегают к использованию чистого SQL. Приведем небольшой пример:
```python
>>> from core.models import Employee
>>> for employee in Employee.objects.raw('SELECT * from core_employee'): 
...  print(employee.first_name)
... 
Джон
Джейн
```

Более подробно о таких запросах написано [в документации](https://docs.djangoproject.com/en/2.0/topics/db/sql/).

## Интерфейс пользователя
Так как в описании модели содержится почти вся информация, необходимая с ее работой, у Django не возникает проблем с автоматической генерацией форм для моделей. Покажем, как Django автоматически сгенерирует нам административную панель, основываясь на этой информации.

В файл `admin.py` добавим следующие строки:
```python
from django.contrib import admin

from . import models

admin.site.register(models.Employee)
admin.site.register(models.Contractor)
admin.site.register(models.Email)
admin.site.register(models.Phone)
admin.site.register(models.Language)
admin.site.register(models.LanguageEmployee)
```

Создадим теперь аккаунт администратора:
```bash
python manage.py createsuperuser
Username (leave blank to use 'USER'): admin
Email address: email@mail.ru
Password: 
Password (again): 
Superuser created successfully.
```
Запустим сервер: `python manage.py runserver`.

Откроем теперь в браузере http://127.0.0.1:8000/admin/ и введем логин и пароль только что сохданного администратора. Теперь мы можем управлять объектами наших моделей через симпатичный GUI-интерфейс (по этому адресу, например, будет расположена форма добавления контрактника: http://127.0.0.1:8000/admin/core/contractor/add/).

## Подключение к MySQL
Бывают [ситуации](https://sqlite.org/whentouse.html), в которых SQLite следует заменить более мощной СУБД.
Для иллюстрации, мы продемонстрируем процесс подключения Django к MySQL.
Мы будем предполагать, что MySQL уже настроена и готова к работе.

В `settings.py` найдем константу `DATABASES` и изменим ее значение:
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'DB_NAME',
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'USER': 'root',
        'PASSWORD': '',
    }
}
```
Здесь:
- `NAME` — имя базы данных, созданной для проекта;
- `HOST` и `PORT` — хост, на котором располагается база данных;
- `USER` и `PASSWORD` — логин и пароль пользователя, под которым разрешено заходить базе данных.

Далее установим одну из следующих библиотек для работы с MySQL: MySQL-python, pymysql, mysqlclient.
```
pip install mysqlclient
```
Если одна из них не заработает, удалим ее (`pip uninstall mysqlclient`) и установим другую.

Теперь можно провести миграции (`python manage.py migrate`). Если во время проведения миграций возникнет ошибка 150, в `settings.py` слеует добавить следующию опцию:
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'DB_NAME',
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'USER': 'root',
        'PASSWORD': '',
        'OPTIONS': {
           "init_command": "SET storage_engine=MyISAM",
        },
    }
}
```
Больше ничего менять не нужно, наш проект продолжит работать, как раньше.

## Дальнейшее чтение


Заинтересовавшимся разработкой сайтов на Django можно порекомендовать следующие ресурсы:
- Интерактивный учебник по синтаксису Python: http://pythontutor.ru/
- Руководство по разработке и запуску блога на Django от DjangoGirls: https://tutorial.djangogirls.org/ru/
- Подробный официальный туториал для начинающих: https://docs.djangoproject.com/en/1.11/intro/
