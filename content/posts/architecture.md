---
title: "Arcitecture"
date: 2021-05-27T19:47:11+05:00
draft: false
---

# Архитектура приложения

### Disclaimer

Я занимаюсь коммерческой  разработкой веб-приложений и микросервисов уже более 10 лет. За годы обучения в университете было много курсов и предметов различной степени полезности: архитектура и внутренее устройство микропроцессоров, сети, низкоуровневая разработка на assembler, высокоуровневая разработка на С++, проектирование баз данных и работа с СУБД, компьютерная графика и куча других вещей, которые как оказалось в реальной работе не очень пригодились. Из более менее полезного были только паттерны проектирования от банды четырех и самые азы unit-тестирования. А вот про то как правильно спроектировать и написать архитектуру приложения, которое легко поддерживать, расширять и тестировать, к сожалению, ничего не было.

Документация к Django ответов на вопросы не давала, статьи в блогах и записи докладов с конференций только поднимали еще больше вопросов. ["Классический доклад"](https://www.youtube.com/watch?v=Nsjsiz2A9mg) Роберта Мартина по теме поначалу так же не зашел, но стало понятно в какую сторону двигаться. Подход, о котором пойдет речь далее, является некоторой компиляцией идей взятых из Чистой Архитектуры, а также [рекомендаций](https://docs.microsoft.com/ru-ru/dotnet/architecture/microservices/architect-microservice-container-applications/) Microsoft по проектированию приложений.

### Начало

Как было раньше. У нас было легаси веб-приложение, которое было наспех переписано c PHP на Python под Django со всеми вытекающими последствиями:

- Бизнес-логика размазана между моделями и контроллерами;
- Запросы в БД через ORM прямо из Django-шаблонов;
- Бесконечные методы на несколько сотен строк, которые занимаются всем на свете: от получения и изменения данных в бд до отправки писем и отрисовки html страниц;
- Полное отсутствие автоматических тестов, потому что во-первых их сложно написать в такой ситуации и они будут выполняться вечность, во-вторых нет времени, ведь бизнесу нужны новые фичи и дедлайном на вчера.

Со временем сложность и стоимость внеднения новой функциональности начала зашкаливать. Одни и те же баги, которые вроде были исправлены в прошлом месяце, начали всплывать снова и снова. Стало понятно что нужно что-то менять.

Методом проб и ошибок получилось прийти к следующей архитектуре. Как и в чистой архитектуре, приложение разделено на слои, каждый из которых отвечает за свою конкретную часть:

- Слой бизнес логики
- Слой доступа к данным
- Слой взаимодействия с внешними сервисами
- Слой внешнего интерфейса самого приложения

Далее пойдет описание каждого из них.

### Бизнес логика

В данном слое мы описываем все бизнес-сущности, с которыми работает наше приложение, а так же всю бизнес-логику. Тут же описываются все интерфейсы/протоколы для взаимодействия с другими слоями. Важно заметить что объекты в этом слое ничего не знают о хранилище и сами в БД для получения или обновления не ходят. На данном уровне мы только описываем интерфейс взаимодействия с хранилищем, конкретная реализация же будет находится в слое доступа к данным.

#### Entity

Entity - объекты бизнес-сущности. Максимально простые, содержат в себе только описание полей и какие то базовые правила бизнес-логики. В python для них удобно использовать объекты из библиотеки [attrs](https://www.attrs.org/en/stable/) или новые [dataclasses](https://docs.python.org/3/library/dataclasses.html), которые появилиьсь в python 3.7.

```
    from dataclasses import dataclass, field
    from typing import Optional

    from passlib.handlers.pbkdf2 import pbkdf2_sha512  # type: ignore


    @dataclass
    class Entity:
        """ Базовый класс для всех бизнес-сущностей."""
        key: int


    @dataclass
    class Permission(Entity):
        name: str


    @dataclass
    class User(Entity):
        """Бизнес-сущность пользователя."""

        email: str
        password: Optional[str] = None
        is_superuser: bool = False
        permissions: list[Permission] = field(default_factory=list)

        def change_password(self, password: str) -> None:
            self.password = pbkdf2_sha512.encrypt(
                password, rounds=10000, salt_size=10
            )

        def verify_password(self, password: str) -> bool:
            try:
                valid = pbkdf2_sha512.verify(password, self.password)
            except ValueError:
                valid = False
            return valid
```

#### Service

Service - объект, содержащий в себе всю бизнес-логику, относящуюся к конкретной бизнес-сущности. Например, создание сущностей, различные валидации и прочие правила бизнес-логики. Данные объекты также ничего не знают о том как данные хранятся в БД, а только имеют доступ к интерфейсу, через который могут их получить или изменить.

```
    from logging import Logger

    from aiohttp_micro.exceptions import EntityAlreadyExist

    from passport.domain import User
    from passport.domain.storage import Storage  # Интерфейс хранилища
    from passport.exceptions import Forbidden


    class UserService:
      """Сервис для работы с пользователем. Реализует основные правила бизнес-логики."""

      def __init__(self, storage: Storage, logger: Logger) -> None:
          self.storage = storage
          self.logger = logger

      async def register(self, email: str, password: str) -> User:
          """Регистрация нового пользователя.

          Args:
            - email: Email нового пользователя
            - password: Пароль пользователя

          Raises:
            - EntityAlreadyExist: Пользователь с таким еmail уже существует

          Return: Возвращает готовый объект пользователя
          """

          exist = await self.storage.users.exists(email)

          if exist:
              raise EntityAlreadyExist()

          user = User(
              key=0, email=email, password="", is_superuser=False, permissions=[]
          )
          user.set_password(password)

          await self.storage.users.add(user)

          self.logger.info('Successfully register user', email=email)

          return user

      async def login(self, email: str, password: str) -> User:
          """Проверка при логине что пользователь существует и ввел правильный пароль.

          Args:
            - email: электронная почта пользователя
            - password: пароль пользователя

          Raises:
            - Forbidden: Пользователь ввел не правильный пароль, доступ запрещен.

          Return: Объект авторизованного пользователя
          """

          user = await self.storage.users.fetch_by_email(email)

          is_valid = user.verify_password(password)
          if not is_valid:
              raise Forbidden()

          return user

      async def fetch(self, key: int) -> User:
          """ Получение объекта пользователя из хранилища.

          Args:
            - key: идентификатор пользователя.

          Return: Объект пользователя.
          """

          return await self.storage.users.fetch_by_key(key)
```

Как видно из примера, объект описывает все необходимые бизнес-правила. Все зависимости он получает из вне и поэтому может быть легко покрыт unit-тестами, которые не требуют подключения к базе данных и будут выполняться быстро и независимо друг от друга.

#### Use Case

Use case - объект содержащий описание сложных бизнес-сценариев, требующих взаимодействия нескольких сервисов. Например, регистрацией пользователя занимается сервис UserService, а отправкой уведомления об этом на почту будет заниматься MailService.

```
    from logging import Logger

    from passport.domain import User
    from passport.domain.storage import Storage  # Интерфейс хранилища
    from passport.services.users import UserService

    class UseCase:
        """Базовый класс для всех сценариев."""
        def __init__(self, storage: Storage, logger: Logger) -> None:
            self.storage = storage
            self.logger = logger


    class MailService:
        """ Сервис отправки уведомлений по почте. """

        def __init__(self, gateway: MailGateway, logger: Logger) -> None:
          self.gateway = gateway
          self.logger = logger

        async def send_registration_email(self, user: User) -> None:
            ...


    class RegisterUserUseCase(UseCase):
        """Сценарий регистрации пользователя."""

        def __init__(self, storage: Storage, mail_gateway: MailGateway, logger: Logger) -> None:
            super().__init__(storage, logger)

            self.user_service = UserService(storage, logger)
            self.mail_service = MailService(mail_gateway, logger)

        async def execute(self, email: str, password: str) -> User:
            """Регистрация пользователя. В случае успеха отправим письмо на почту.

            Args:
              - email: электронная почта нового пользователя.
              - password: пароль нового пользователя.

            Return: Готовый объект зарегистрированного пользователя.
            """

            user = await self.user_service.register(email, password)

            await self.mail_service.send_registration_email(user=user)

            return user
```

Use case-объекты также ничего не знают о конкретной реализации ни хранилища, ни того каким образом отправляются уведомления. Логика тут довольно простая и так же может быть легко покрыта unit-тестами.


### Слой доступа к данным

Данный слой содержит в себе всю логику того как нам хранить и модифицировать наши данные. Здесь будет находиться описание структуры моделей в случае использования ORM вроде SQLAlchemy или просто таблиц в случае использования [databases](https://github.com/encode/databases) для асинхронного подхода. Вся логика доступа к моделям и таблицам находится в объектах, реализующих интерфейс Repository из слоя бизнес-логики.

#### Repository

Repository - объект, содержащий всю логику по доступу и модификации конкретных моделей.

Пример интерфейса репозитория для работы с пользователями:

```
    # src/passport/domain/storage/users.py

    from typing import Protocol

    from passport.domain import Permission, User


    class UsersRepo(Protocol):
        """Базовый протокол для описания операций получения и модификации бизнес-сущности пользователя."""

        async def fetch_by_key(self, key: int) -> User:
            ...

        async def fetch_by_email(self, email: str) -> User:
            ...

        async def exists(self, email: str) -> bool:
            ...

        async def add(self, user: User) -> None:
            ...

        async def add_permission(self, user: User, permission: Permission) -> None:
            ...

        async def remove_permission(
            self, user: User, permission: Permission
        ) -> None:
            ...

        async def save_user(self, email: str, password: str) -> int:
            ...
```

Пример конкретной реализации интерфейса для работы с пользователями, полный листинг [тут](https://github.com/clayman-micro/passport/blob/master/src/passport/storage/users.py):

```
    from datetime import datetime

    import sqlalchemy  # type: ignore
    from aiohttp_micro.exceptions import EntityNotFound  # type: ignore
    from aiohttp_storage.storage import metadata  # type: ignore
    from databases import Database
    from sqlalchemy import func
    from sqlalchemy.orm.query import Query  # type: ignore

    from passport.domain import Permission, User
    from passport.domain.storage.users import UsersRepo


    users = sqlalchemy.Table(
        "users",
        metadata,
        sqlalchemy.Column("id", sqlalchemy.Integer, primary_key=True),
        sqlalchemy.Column(
            "email", sqlalchemy.String(255), nullable=False, unique=True
        ),
        sqlalchemy.Column("password", sqlalchemy.String(255), nullable=False),
        sqlalchemy.Column("is_active", sqlalchemy.Boolean, default=True),
        sqlalchemy.Column("is_superuser", sqlalchemy.Boolean, default=False),
        sqlalchemy.Column(
            "last_login", sqlalchemy.DateTime, default=datetime.utcnow
        ),
        sqlalchemy.Column(
            "created_on", sqlalchemy.DateTime, default=datetime.utcnow
        ),
    )

    class UsersDBRepo(UsersRepo):
        def __init__(self, database: Database) -> None:
            self._database = database

        def get_query(self) -> Query:
            return sqlalchemy.select(
                [users.c.id, users.c.email, users.c.password]
            ).where(
                users.c.is_active == True  # noqa: E712
            )

        def _process_row(self, row) -> User:
            return User(
                key=row["id"], email=row["email"], password=row["password"]
            )  # type: ignore

        async def fetch_by_key(self, key: int) -> User:
            query = self.get_query().where(users.c.id == key)
            row = await self._database.fetch_one(query)

            if not row:
                raise EntityNotFound()

            return self._process_row(row)

        async def fetch_by_email(self, email: str) -> User:
            query = self.get_query().where(users.c.email == email)
            row = await self._database.fetch_one(query)

            if not row:
                raise EntityNotFound()

            return self._process_row(row)

        async def exists(self, email: str) -> bool:
            query = sqlalchemy.select([func.count(users.c.id)]).where(
                users.c.email == email
            )
            count = await self._database.fetch_val(query)

            return count > 0

        ...
```

Данные объекты так же могут быть легко покрыты unit-тестами, использующими как реальную БД, так и Mock-заглушку переданную в виде зависимости.

#### Storage

Storage - данный объект не обязателен и добавлен для удобства. Описывает все виды репозиториев приложения и позволяет сэкономить на передаче параметров в объекты бизнес логики.

```
    # src/passport/domain/storage/__init__.py - интерфейс в бизнес-логике

    from abc import ABC

    from passport.domain.storage.sessions import SessionRepo
    from passport.domain.storage.users import UsersRepo


    class Storage(ABC):
        sessions: SessionRepo
        users: UsersRepo


    # src/passport/storage/__init__.py - конкретная реализация

    from aiohttp_storage.storage import (  # type: ignore
        DBStorage as AbstractDBStorage,
    )
    from databases import Database

    from passport.domain.storage import Storage
    from passport.storage.sessions import SessionDBStorage
    from passport.storage.users import UsersDBRepo


    class DBStorage(Storage, AbstractDBStorage):
        def __init__(self, database: Database) -> None:
            super().__init__(database=database)

            self.sessions = SessionDBStorage(database=database)
            self.users = UsersDBRepo(database=database)
```


### Слой взаимодействия с внешними сервисами

По тому же принципу как мы отделяем слой доступа к данным, можно выделить отдельно конкретные реализации объектов, взаимодействующими с другими компонентами системы. Например: отправка уведомлений по email или через Telegram бота, получение данных о счетах и балансе пользователя из микросервиса биллинга, отправка уведомлений пользователю в браузер через сервис push-нотификаций. Объекты реализуют интерфейс Gateway из слоя бизнес-логики.


### Внешний интерфейс приложения

В данном слое уже находится все что относится ко взаимодействию с нашим приложеним из внешнего мира. Controller/handler из веб-фреймворка, схемы валидации http запросов и сериализации ответов, команды для CLI, задания для Celery. Поскольку вся бизнес логика у нас уже описана в UseCase, тут нам достаточно просто собрать все воедино и запустить выполнение

#### Handlers/Controllers

Обработчик POST запроса на примере библиотеки [aiohttp](https://docs.aiohttp.org/en/stable/)

```
    from aiohttp import web
    from aiohttp_micro.exceptions import EntityNotFound  # type: ignore
    from aiohttp_micro.handlers import (  # type: ignore
        json_response,
        validate_payload,
    )
    from marshmallow import fields, Schema

    from passport.exceptions import Forbidden
    from passport.handlers import CredentialsPayloadSchema, session_required
    from passport.storage import DBStorage
    from passport.use_cases.users import LoginUseCase


    class CredentialsPayloadSchema(Schema):
        email = fields.Str(required=True, description="User email")
        password = fields.Str(required=True, description="User password")


    @validate_payload(CredentialsPayloadSchema)
    async def login(payload: Dict[str, str], request: web.Request) -> web.Response:
        use_case = LoginUseCase(app=request.app)

        try:
            user = await use_case.execute(payload["email"], payload["password"])
        except Forbidden:
            raise web.HTTPForbidden()
        except EntityNotFound:
            raise web.HTTPNotFound()

        config = request.app["config"]

        session_key = secrets.token_urlsafe(32)
        expires = datetime.now() + timedelta(days=config.sessions.expire)

        storage = DBStorage(database=request.app["db"])
        await storage.sessions.add(user, session_key, expires)

        request.app["logger"].info("User logged in", user=user.email)

        response = json_response({})
        response.set_cookie(
            name=config.sessions.cookie,
            value=session_key,
            max_age=config.sessions.expire * 24 * 60 * 60,
            domain=config.sessions.domain,
            httponly="True",
        )

        return response
```

### Заключение

Таким образом мы получили простую с точки зрения организации и разделения отвественностей архитектуру, которую достаточно просто поддерживать и развивать. Нет проблем с покрытием кода автоматическим unit и интеграционными тестами, которые будут выполняться быстро и не зависимо друг от друга. Вот примеры некоторых сервисов, реализованных с использованием описанного подхода:

  - [Passport](https://github.com/clayman-micro/passport) - простой микросервис регистрации и авторизации пользователей.
  - [Wallet](https://github.com/clayman-micro/wallet/tree/release/2.4.0) - сервис учета и анализа личных доходов и расходов, пока еще находится в стадии разработки)

Описаный подход не претендует на ультимативность, но ничего лучше пока придумать не удалось 😅.
Спасибо за внимание.
