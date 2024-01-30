Глава 7. Динамическое определение и переключение языка в Aiogram
-----------------------------------------------------------------

Практический пример
~~~~~~~~~~~~~~~~~~~

Теперь мы знаем, как проводится интернационализация и локализация
продукта. Но нам бы хотелось это все внедрить в интерфейс нашего
приложения на фреймворке Aiogram, чтобы повысить качество UX (user
experience).

Пример будет реализован с помощью Fluent, поскольку gettext уже изучен и
по факту является промышленным стандартом. А нам хочется попробовать
что-то новое.

Спроектируем взаимодействие с пользователем следующим образом.

1. Если пользователь впервые начал взаимодействовать с ботом, то мы не
   знаем на каком языке он говорит. Попытаемся получить язык
   пользователя из апдейта. Но эта опция зависит от клиента Telegram,
   которым пользуется пользователь, и часто это поле не заполнено. В
   таком случае отдадим какой-то язык по умолчанию, например,
   английский.

2. Если пользователь выбрал в меню бота или отправил команду выбора
   языка, то сохраним язык в базе данных и переключим язык. В
   дальнейшем, при взаимодействии, будем использовать язык из базы
   данных.

3. Нам нужно сделать переводы для всех элементов интерфейса, включая
   сообщения, клавиатуры, меню, алерты.

4. В каких-то случаях при отправке картинок или документов —
   локализовать картинки или документы.

В учебных целях мы реализуем следующие вещи:

-  Наш бот будет стартовать по команде ``/start``
-  выводить текст помощи по команде ``/help`` или после получения слова
   ``помощь`` или ``help``, в зависимости от текущего языка.
-  Отдавать клавиатуру на нужном языке.
-  По команде ``/languge_en`` и ``/languge_ru`` переключаться на
   соответствующий язык.
-  По команде ``/photo`` отправлять картинку, переведенную на текущий
   язык.

Нам потребуется база данных, для хранения языка пользователя. В коде
будет минимальный пример, просто "чтоб работало", а также много
сообщений и много контекста. Ну и, как ранее говорилось, будет много
дублирования кода - здравствуй WET, прощай DRY.

Структура части проекта, отвечающая за интернационализацию и
локализацию, будет примерно такой:

::

   ...
   ├───database
   │       bot.db
   │       database.py
   ├───locales
   │   ├───en
   │   │   ├───LC_MESSAGES
   │   │   │       messages.ftl
   │   │   └───static
   │   │           bayan_en.jpg
   │   └───ru
   │       ├───LC_MESSAGES
   │       │       messages.ftl
   │       └───static
   │               bayan_ru.jpg
   └───middlewares
           db_middleware.py
           i18n_middleware.py
   ...

База данных, интерфейс к ней, папка с переводами и локализованными
картинками, и два внешних middleware для работы с базой данных и
переводами.

Базу данных и интерфейс к ней вы уже научились делать. Интерфейс БД
должен уметь добавлять пользователя и получать его язык. Этого
достаточно для нашего примера.

Самое главное, это то, что классы движка перевода, реализованные в
Aiogram, не позволяют из коробки реализовать задуманный нами функционал.
В главе Конфигурация движка перевода мы уже приводили описание этих
классов. Напомню:

``SimpleI18nMiddleware`` - выбирает код языка из объекта User,
полученного в событии. Однако не все клиенты Telegram отдают это
значение. Очень часто объект language_code не заполнен и является пустой
строкой.

``ConstI18nMiddleware`` - выбирает статически определенную локаль. Это
не динамично и скучно.

``FSMI18nMiddleware`` - хранит локаль в хранилище FSM. Но у нас ее там
пока нет.

И у нас есть то, что нужно: ``I18nMiddleware``. Это базовый абстрактный
класс для наследования и создания собственного обработчика.

Создадим в нашем приложении объект этого класса, и передадим туда наш
кастомный менеджер, который реализуем ниже.

.. code-block:: python

   async def main() -> None:
       ...
       dp = Dispatcher()
       dp.include_router(router)

       i18n = I18nMiddleware(
           core=FluentRuntimeCore(
               path="locales/{locale}/LC_MESSAGES"
           ),

           # передаем наш кастомный менеджер языка из middlewares/i18n_middleware.py
           manager=i18n_middleware.UserManager(),
           default_locale="en")
       ...

Начнем с реализации своего менеджера. Создадим файл
``middlewares/i18n_middleware.py``.

.. code-block:: python

   from aiogram_i18n.managers import BaseManager
   from aiogram.types.user import User
   from database.database import Database


   class UserManager(BaseManager):
       """
       Собственная реализация middleware - менеджера для интернационализации
       на базе класса BaseManager из библиотеки aiogram_i18n.
       BaseManager имеет абстрактные методы set_locale и get_locale, которые
       нам нужно реализовать. Кроме того, при инициализации объекта класса,
       выполняются LocaleSetter и LocaleGetter (см. реализацию BaseManager).

       P.S. В случае использования gettext необходимо проверить реализацию
       класса, так как не gettext не тестировалось
       """

       async def get_locale(self, event_from_user: User, db: Database = None) -> str:
           default = event_from_user.language_code or self.default_locale
           if db:
               user_lang = db.get_lang(event_from_user.id)
               if user_lang:
                   return user_lang
           return default
       async def set_locale(self, locale: str, event_from_user: User, db: Database = None) -> None:
           if db:
               db.set_lang(event_from_user.id, locale)

По сути мы просто реализовали два метода:

``get_locale`` – геттер, который сначала проверяет есть ли в базе данных
у пользователя какой-то язык. Если в базе ничего нет, то пытается
получить язык из клиента, и если и его нет - просто отдает локаль
по-умолчанию.

``set_locale`` – сеттер, который просто записывает язык в базу данных, а
если базы нет, то ничего не делает.

Естественно, эту логику работы с языком каждый придумывает себе сам под
свои задачи и особенности работы и используемые инструменты (кэш,
хранилище и т.п.).

Регистрируем middleware сначала для базы данных, а затем i18n. Не
забываем, что у i18n есть метод .setup(), который правильно регистрирует
этот middleware.

.. code-block:: python

   async def main() -> None:
       ...
       # Регистрация middleware.
       # Сначала регистрируется middleware для базы данных, так как там хранится язык.
       dp.update.outer_middleware.register(db_middleware.DBMiddleware())

       # Регистрируем i18n middleware
       i18n.setup(dispatcher=dp)
       ...

Сначала пропишем наши хэндлеры. А уже в конце займемся переводами.
Импорты:

.. code-block:: python

   from aiogram_i18n import I18nContext, LazyProxy, I18nMiddleware
   from aiogram_i18n.cores.fluent_runtime_core import FluentRuntimeCore

   from aiogram_i18n.types import (
       ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove
       # you should import mutable objects from here if you want to use LazyProxy in them
   )

Первый хэндлер обрабатывает команду ``/start`` и сохраняет пользователя
в БД. Язык нам не известен, поэтому его мы не сохраняем.

.. code-block:: python


   @router.message(CommandStart())
   async def process_start_command(message: Message, i18n: I18nContext, db: Database):
       if not db.get_user(message.from_user.id):
           db.add_user(message.from_user.id, message.from_user.username)
       name = message.from_user.full_name
       await message.answer(text=i18n.get("hello", user=name, language=i18n.locale),
                            reply_markup=rkb
                            )

Следующий хэндлер обрабатывает команду ``/help`` и слова ``help``,
``Help``, ``помощь``, ``Помощь``, введенные на родном языке
пользователя. Поскольку на момент попадания в фильтрацию объект i18n
middleware не вызывается, язык мы не можем получить. Поэтому используем
ленивую подстановку текстов ``LazyProxy``. Мутабельные объекты, например
клавиатуры, для LazyProxy экспортируем не из основной библиотеки
aiogram, а из ``aiogram_i18n``.

.. code-block:: python

   @router.message(Command("help"))
   @router.message(F.text == LazyProxy("help", case="capital"))
   @router.message(F.text == LazyProxy("help", case="lower"))
   async def cmd_help(message: Message, i18n: I18nContext) -> Any:
       return message.reply(text=i18n.get("help-message"))

Создадим хэндлер для команды обработки смены языка.

.. code-block:: python

   async def switch_language(message: Message, i18n: I18nContext, locale_code: str):
       await i18n.set_locale(locale_code)
       await message.answer(i18n.get("lang-is-switched"), reply_markup=rkb)


   @router.message(Command("language_en"))
   async def switch_to_en(message: Message, i18n: I18nContext) -> None:
       await switch_language(message, i18n,"en")


   @router.message(Command("language_ru"))
   async def switch_to_en(message: Message, i18n: I18nContext) -> None:
       await switch_language(message, i18n,"ru")

Мы видим дублирование кода, но это неизбежно. Часть было вынесено в
функцию switch_language().

Далее отправка изображения. Изображения будут лежать в
``locale/имя_локали/static/имя_картинки_локаль.jpg``.

.. code-block:: python

   @router.message(Command("photo"))
   @router.message(F.text == LazyProxy("photo"))
   async def sent_photo(message: Message, i18n: I18nContext) -> None:
       locale_code = i18n.locale
       path_to_photo = f"locales/{locale_code}/static/my_image_{locale_code}.jpg"
       await message.answer_photo(photo=FSInputFile(path_to_photo))

Следующий хэндлер отвечает за обработку остальных сообщений. При этом он
после ответ выдает еще и дату сообщения в формате, специфичном для
локали пользователя. То есть "День Месяц Год" или "Month Day, Year".

.. code-block:: python

   @router.message()
   async def handler_common(message: Message, i18n: I18nContext) -> None:
       await message.answer(text=i18n.get("i-dont-know"))
       await message.answer(text=i18n.get("show-date", date_=message.date))

Ну и клавиатура, которую мы импортировали из ``aiogram_i18n.types``

.. code-block:: python

   # Это тестовая клавиатура
   rkb = ReplyKeyboardMarkup(
       keyboard=[
           [KeyboardButton(text=LazyProxy("help", case="capital"))]  # or L.help()
       ], resize_keyboard=True
   )

Текст клавиатуры будет также лениво переведен в момент отправки
сообщения, когда уже язык будет известен.

Осталось сделать саму локализацию. Складываем картинки в папки локалей.
И создаем файлы переводов в формате .ftl в соответствующих папках.
Логика работы описана в комментариях в каждом файле.

Английский перевод:

.. code-block:: fluent

   # This Source Code Form is subject to the terms of the Mozilla Public
   # License, v. 2.0. If a copy of the MPL was not distributed with this
   # file, You can obtain one at http://mozilla.org/MPL/2.0/.
   # Это был пример лицензии

   ### Файл примера перевода на английский язык
   ### Логика перевода изменится, не затрагивая код и другие переводы
   ### С тройного шарпа начинается комментарий уровня файла

   ## Это комментарий уровня группировки блоков в тексте. См. документацию.
   ## Hello section

   # Это пример термина. Термин начинается с дефиса.
   # Посмотрите как это работало в русском переводе. Здесь же мы изменим логику.
   # Падежи нам не нужны, но может потребоваться притяжательная форма
   -telegram = { $case ->
        *[common] Telegram
         [possessive] Telegram's
       }

   # { $user } - user name, { $language } - language code.
   # Это было описание переменных, которые попадают сюда из основного кода приложения.
   # Термин мы берем из этого же файла перевода,
   # и вставляем с параметром нужного контекста использования (в нашем случае падежа).
   hello = Hi, <b>{ $user }</b>!
       { $language ->
        [None] In your { -telegram(case: "common") } client a language isn't set.
               Therefore, everything will be displayed in default language.
       *[any] Your Telegram client is set to { $language }. Therefore, everything will be displayed in this language.
       }

   help = { $case ->
       *[capital] Help
        [lower] help
       }
   help-message =
       <b>Welcome to the bot.</b>
       Our bot can't do anything useful, but it can switch languages with dexterity.

       The following commands are available in the bot:
       /start to start working with the bot.
       /help or just send the word <b><i>help</i></b> to show this message.
       /language_en { switch-to-en }
       /language_ru { switch-to-ru }
       /photo or just send the word <b><i>photo</i></b> to send photo to you.


   # { $language } - language code.
   # The current language is { $language }.
   cur-lang = The current language is: <i>{ $language }</i>

   ## Switch language section

   # Название языка мы отображаем на родном языке, чтоб человек увидел знакомые буквы и поонял, что не все потеряно.
   en-lang = English
   ru-lang = Русский
   switch-to-en = Switch the interface to { en-lang }.
   switch-to-ru = Switch the interface to { ru-lang }.
   lang-is-switched = Display language is { en-lang }.

   photo = photo

   ## Common messages section

   i-dont-know = I'm so stupid bot. Make me clever.
   show-date = But look! Pretty date on English: { DATETIME($date_, month: "long", year: "numeric", day: "numeric", weekday: "long") }

Русский перевод:

.. code-block:: fluent

   # This Source Code Form is subject to the terms of the Mozilla Public
   # License, v. 2.0. If a copy of the MPL was not distributed with this
   # file, You can obtain one at http://mozilla.org/MPL/2.0/.
   # Это был пример лицензии

   ### Файл примера перевода на русский язык
   ### Важно. Не забудь полить помидоры...
   ### С тройного шарпа начинается комментарий уровня файла

   ## Это комментарий уровня группировки блоков в тексте. См. документацию.
   ## Hello section

   # Это пример термина. Термин начинается с дефиса.
   # Термины можно передавать внутри сообщений, указывая переменные для параметризации в скобках.
   # То есть это как атрибуты, но мы их задаем в тексте переводов, а не получаем извне.
   # Мы будем издеваться над языком, чтобы увидеть как и что работает
   -telegram = {$case ->
       *[nominative] Телеграм {"{"}Telegram{"}"}
        [genitive] Телеграма ({"{"}Telegram'а{"}"})
        [dative] Телеграму ({"{"}Telegram'у{"}"})
        [accusative] Телеграм ({"{"}Telegram{"}"})
        [instrumental] Телеграмом ({"{"}Telegram'ом{"}"})
        [prepositional] Телеграме ({"{"}Telegram'е{"}"})
       }
   # {"}"}  это пример экранированного символа.
   # Падежи
   # nominative - именительный
   # genitive - родительный
   # dative - дательный
   # accusative - винительный
   # instrumental творительный.
   # prepositional - предложный

   # { $user } - user name, { $language } - language code.
   # Это было описание переменных, которые попадают сюда из основного кода приложения.
   # Термин мы берем из этого же файла перевода,
   # и вставляем с параметром нужного контекста использования (в нашем случае падежа).
   hello = Привет, <b>{ $user }</b>!
           У тебя в клиенте { -telegram(case: "nominative") } { $language ->
        [None] не указан язык, поэтому все будет отображается на языке по-умолчанию.
       *[any] указан язык { $language }, поэтому все будет отображается на этом языке.
       }

   # а так мы вставляем символы unicode по номеру \uHHHH. Например,
   # tears-of-joy1 = {"\U01F602"}
   # tears-of-joy2 = 😂

   help = { $case ->
       *[capital] Помощь
        [lower] помощь
       }

   help-message =
       <b>Добро пожаловать в бота.</b>
       Наш бот не умеет ничего полезного, однако с ловкостью может переключать язык.

       В боте доступны следующие команды:
       /start чтобы начать работать с ботом
       /help или просто отправьте слово <b><i>помощь</i></b>, чтобы показать это сообщение
       /language_en { switch-to-en }
       /language_ru { switch-to-ru }
       /photo или просто отправьте слово <b><i>фото</i></b>, чтобы прислать вам картинку


   # Это комментарий подсказка для переводчиков (чтобы не искать что значат эти переменные в коде,
   # который не факт ,что они получат, а если и получат, то не поймут:

   # { $language } - language code.
   # The current language is { $language }.
   cur-lang = Текущий язык: <i>{ $language }</i>

   ## Switch language section

   en-lang = English
   ru-lang = Русский
   switch-to-en = Переключить интерфейс на { en-lang } язык.

   # В фигурных скобках пример интерполяции одного сообщения в другом.
   switch-to-ru = Переключить интерфейс на { ru-lang } язык.
   lang-is-switched = Язык переключен на { ru-lang }.

   photo = фото

   ## Common messages section

   i-dont-know = Я тупой бот. Сделай меня умным.
   show-date = Но посмотри! Красивая дата по правилам Русского языка: { DATETIME($date_, month: "long", year: "numeric", day: "numeric", weekday: "long") }

Запускаем, тестируем.

Исправляем ошибки.
~~~~~~~~~~~~~~~~~~

Наш проект серьезно усложнился и нужно произвести тестирование и
отладку. Вот некоторые ошибки, которые часто возникают.

**aiogram_i18n.exceptions.NoTranslateFileExistsError: files with
extension (.ftl) in folder (locales/ru/LC_MESSAGES) not found** — ошибка
возникает когда файл перевода не найден по указанному нами пути.

**KeyNotFoundError: Key ‘enter-a-number’ not found** — ошибка возникает,
когда в коде есть ключ, а в переводе его нет. Например, вызываем
``i18n.get("help")``, а такой строчки ``help`` нет в файле перевода
соответствующего языка.

**fluent.runtime.errors.FluentReferenceError: Unknown external: user** —
такая ошибка возникает, когда вы забываете передать в вызове функции
основного кода нужный аргумент для ключа. Например, в нашем переводе
есть такое сообщение:

.. code-block:: fluent

   hello = Привет, <b>{ $user }</b>!
       У тебя в клиенте { -telegram(case: "nominative") } { $language ->
       [None] не указан язык, поэтому все будет отображается на языке по-умолчанию.
      *[any] указан язык { $language }, поэтому все будет отображается на этом языке.
       }

Здесь ``{ -telegram }`` – это термин. И он управляется
конструкцией\ ``(case: "nominative")`` только внутри языкового файла. А
вот дальше используются аргументы ``{ $user }`` и ``{ $language }``,
которые нужно передать из основного кода. Мы их передаем как именованные
аргументы:

.. code-block:: python

   await message.answer(text=i18n.get("hello", user=name, language=i18n.locale))

или еще возможен такой способ:

.. code-block:: python

   await message.answer(text=i18n.hello(user=name, language=i18n.locale))

Так вот в случае, если мы неправильно указали аргумент в основном коде
(например ``nmae`` вместо ``name``) или вообще не указали, то во время
компиляции перевода отсутствие аргумента и вызывает ошибку.
