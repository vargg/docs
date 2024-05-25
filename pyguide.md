# Руководство по стилю Python ([src](https://github.com/google/styleguide/blob/gh-pages/pyguide.md))

Это руководство по стилю представляет собой список того, как можно и как нельзя оформлять Python-код.

## 1. Правила языка python

### 1.1 pylint

pylint — это инструмент для поиска ошибок и проблем со стилем в исходном коде Python. Он находит проблемы, которые обычно обнаруживаются компилятором менее динамичных языков, таких как C и C++.

pylint позволяет отлавливать простые ошибки: опечатки, использование переменных перед присваиванием и т.д.

В pylint каждое предупреждение идентифицируется символическим именем (empty-docstring).

При этом pylint — не серебряная пуля. Иногда требуется отклоняться от его стандартных правил. В таких случаях можно отключить предупреждение для отдельного участка кода. Для этого следует установить комментарий на уровне строки:

```python
def do_PUT(self):  # WSGI name, so pylint: disable=invalid-name
    ...
```

Следует использовать формат `pylint: disable`; формат `pylint: disable-msg` является устаревшим. Если причина подавления ошибки не ясна из символического названия, следует дополнительно указать пояснение.

Подавленные таким способом ошибки можно впоследствии легко отыскать и вернуться к ним.

Вы можете посмотреть список всех существующих pylint-предупреждений, выполнив:

```bash
$ pylint --list-msgs
```

Чтобы получить дополнительную информацию о конкретном сообщении, используйте:

```bash
$ pylint --help-msg=invalid-name
```

Предупреждений о неиспользуемых аргументах функции можно избежать, удалив переменные в начале функции. Всегда оставляйте комментарий, объясняющий, почему вы его удаляете.

```python
def viking_cafe_order(spam: str, beans: str, eggs: str | None = None) -> str:
    del beans, eggs  # Unused by vikings.
    return spam + spam + spam
```

Другие распространенные формы подавления этого предупреждения включают использование `_` в качестве идентификатора неиспользуемого аргумента, добавление к имени аргумента префикса `unused_` или последующее присвоение переменной `_`. Такие подходы нежелательны: они создают неудобства при передаче аргументов по имени и не гарантируют, что аргументы фактически не используются.

### 1.2 Импорты

Предпочтительно использовать `import` для целых пакетов и модулей, а не для отдельных типов, классов или функций. Исключения — модули `typing`, `typing_extensions`, `collections.abc` (см. [импорт аннотаций типов](#21912-импорт-аннотаций-типов)).

- используйте `import x` для импорта пакетов и модулей
- используйте `from x import y`, где `x` — путь до модуля, `y` — имя модуля
- используйте `from x import y as z` в любом из следующих случаев:
    - необходимо импортировать два модуля с одинаковым именем `y`
    - `y` конфликтует с именем верхнего уровня, определенным в текущем модуле
    - `y` конфликтует с каким либо именем параметра, которое является частью общедоступного API
    - `y` — неудобно длинное имя
    - `y` слишком общий в контексте вашего кода (например, `from storage.file_system import options as fs_options`)
    - для распространённых устоявшихся сокращений (например, `import numpy as np`)

Не используйте относительные имена при импорте. Даже если модуль находится в том же пакете, используйте полное имя пакета. Это помогает предотвратить непреднамеренный импорт пакета дважды.

```python
# YES:
# Reference absl.flags in code with the complete name (verbose).
import absl.flags
from doctor.who import jodie

_FOO = absl.flags.DEFINE_string(...)

# Reference flags in code with just the module name (common).
from absl import flags
from doctor.who import jodie

_FOO = flags.DEFINE_string(...)

# NO:
# Unclear what module the author wanted and what will be imported.  The actual
# import behavior depends on external factors controlling sys.path.
# Which possible jodie module did the author intend to import?
import jodie
```

### 1.3 Исключения (exceptions)

Исключения позволяют прервать нормальную последовательность действий в потоке управления при возникновении исключительной ситуации.

Использование исключений может усложнить понимание событий в потоке управления. Так же легко пропустить необходимость обработки исключения при работе с библиотеками.

Исключения должны соответствовать следующим условиям:

- используйте встроенные классы исключений, когда это имеет смысл; например, `ValueError` чтобы указать на программную ошибку (нарушение условий для аргументов функции и т.д.)

- не используйте `assert` вместо условий; `assert` не должны влиять на логику приложения; критерием допустимости будет то, что `assert` можно удалить, не нарушив логику кода

    ```python
    # YES:
    def connect_to_next_port(self, minimum: int) -> int:
        """Connects to the next available port.

        Args:
            minimum: A port value greater or equal to 1024.

        Returns:
            The new minimum port.

        Raises:
            ConnectionError: If no available port is found.
        """
        if minimum < 1024:
            # Note that this raising of ValueError is not mentioned in the doc
            # string's "Raises:" section because it is not appropriate to
            # guarantee this specific behavioral reaction to API misuse.
            raise ValueError(f'Min. port must be at least 1024, not {minimum}.')

        port = self._find_next_open_port(minimum)
        if port is None:
            raise ConnectionError(f'Could not connect to service on port {minimum} or higher.')

        # The code does not depend on the result of this assert.
        assert port >= minimum, (f'Unexpected port {port} when minimum was {minimum}.')

        return port

    # NO:
    def connect_to_next_port(self, minimum: int) -> int:
        """Connects to the next available port.

        Args:
        minimum: A port value greater or equal to 1024.

        Returns:
        The new minimum port.
        """
        # The following code depends on the previous assert.
        assert minimum >= 1024, 'Minimum port must be at least 1024.'

        port = self._find_next_open_port(minimum)
        # The type checking of the return statement relies on the assert.
        assert port is not None

        return port
    ```

- библиотеки или пакеты могут определять свои собственные исключения; при этом они должны наследоваться от существующего исключения; имена исключений должны заканчиваться на `Error` и не должны содержать повторений (`foo.FooError`)

- `except:` улавливает абсолютно всё — вызовы `sys.exit()`, прерывания `Ctrl+C`, сбои unit-тестов и любые другие исключения, которые вы, как правило, не намерены перехватывать; не используйте пустой `except:` или `except Exception:`, кроме случаев
    - последующий reraise исключения
    - создание точки изоляции в программе, дальше которой никакие исключения не распространяются, вместо этого логируются и подавляются

- минимизируйте количество кода в блоке `try — except`; чем больше тело `try`, тем больше вероятность того, что будет перехвачено исключение, вызванное строкой кода, исключения от которой не ожидалось; это может привести к сокрытию непроработанных ошибок и осложнить отладку

- используйте блок `finall` для выполнения кода независимо от того, возникло исключение в блоке `try` или нет

### 1.4 Изменяемое глобальное состояние

Изменяемое глобальное состояние — любая переменная на уровне модуля или атрибут класса (`ClassVar`), которые могут изменяться в процессе выполнения программы.

Следует избегать изменяемых глобальных состояний.

В тех редких случаях, когда использование глобального состояния оправдано, изменяемые глобальные сущности следует объявлять на уровне модуля или как атрибут класса и обозначать приватными, добавляя префикс `_` к имени. При необходимости, внешний доступ к изменяемому глобальному состоянию должен осуществляться через общедоступные функции или методы класса.

Константы уровня модуля разрешены и поощряются. Например: `_MAX_HOLY_HANDGRENADE_COUNT = 3` для константы внутреннего использования или `SIR_LANCELOTS_FAVORITE_COLOR = "blue"` для константы общедоступного API. Константы должны быть названы с использованием заглавных букв и подчеркиваний.

### 1.5 Вложенные / локальные / внутренние классы и функции

Класс может быть определен внутри метода, функции или другого класса. Функция может быть определена внутри метода или функции. Вложенные функции имеют доступ (только для чтения) к переменным, определенным в охватывающих областях.

Такой подход позволяет определять специальные классы и функции, которые будут использоваться только в очень ограниченной области (см. [ADT](https://en.wikipedia.org/wiki/Abstract_data_type)). Часто используется для реализации декораторов.

В то же время вложенные функции и классы могут усложнять написание тестов. Вложенность может сделать охватывающую функцию длиннее и менее читабельной.

Допускается использовать с некоторыми оговорками. Основное назначение — использование некого локального значения через замыкание (не `self` или `cls`).

### 1.6 Включения и генераторные выражения

`list`-включения, `dict`-включения и `set`-включения (comprehensions), а также генераторные выражения предоставляют собой краткий и эффективный способ создания контейнеров и итераторов, не прибегая к использованию традиционных циклов, а так же `map()`, `filter()` и т.д.

Включения — предпочтительный способ для простых случаев.

Простые включения проще, удобнее и читабельнее, чем другие методы. Выражения-генераторы могут быть очень эффективными, поскольку исключают создание контейнера.

Не допускается использование нескольких вложенных `for` или `if`. Следует ориентироваться на удобство чтения и понимания, а не на краткость.

```python
# YES:
result = [mapping_expr for value in iterable if filter_expr]

result = [
    is_valid(metric={'key': value})
    for value in interesting_iterable
    if a_longer_filter_expression(value)
]

descriptive_name = [
    transform({'key': key, 'value': value}, color='black')
    for key, value in generate_iterable(some_input)
    if complicated_condition_is_met(key, value)
]

result = []
for x in range(10):
    for y in range(5):
        if x * y > 10:
            result.append((x, y))

return {
    x: complicated_transform(x)
    for x in long_generator_function(parameter)
    if x is not None
}

return (x**2 for x in range(10))

unique_names = {user.name for user in users if user is not None}

# NO:
result = [(x, y) for x in range(10) for y in range(5) if x * y > 10]

return (
    (x, y, z)
    for x in range(5)
    for y in range(5)
    if x != y
    for z in range(5)
    if y != z
  )
```

### 1.7 Итераторы по-умолчанию

Встроенные контейнеры языка, такие как `dict` и `list`, предоставлют итераторы по-умолчанию и поддерживают операторы проверки вхождения (`in`, `not in`). Они просты и эффективны, выполняют операцию напрямую, без дополнительных вызовов методов.

Предпочтительны во всех случаях, за исключением ситуаций, когда требуется изменять контейнер во время итерации по нему.

```python
# YES:
for key in a_dict: ...
if obj in a_list: ...
for line in a_file: ...
for k, v in a_dict.items(): ...

# NO:
for key in adict.keys(): ...
for line in afile.readlines(): ...
```

### 1.8 Генераторы

Генератор использует меньше памяти, чем функция, которая отдаёт сразу весь список значений. В то же время локальные переменные в генераторе сохраняются и не будут уничтожены сборщиком мусора до тех пор, пока генератор не будет полностью израсходован или сам не будет собран.

В строке документации функций-генераторов следует указывать «Yields:» вместо / дополнительно к «Returns:».

Если генератор управляет дорогостоящим ресурсом, обязательно выполняйте принудительную очистку. Хороший способ выполнить очистку — обернуть генератор контекстным менеджером ([PEP-533](https://peps.python.org/pep-0533/)).

### 1.9 Лямбда-функции

lambda — анонимная функция.

В сравнении с обычными функциями:

- в некоторых ситуациях удобнее для простых действий
- код становится сложнее читать и отлаживать — из-за отсутствия имени труднее понять трассировку стека
- выразительность ограничена, поскольку функция может содержать только выражение

Вместо `map()` или `filter()` с lambda предпочтительнее использовать генераторные выражения.
Если код внутри лямбда-функции занимает более 60–80 символов — лучше определить его как обычную функцию.

Для распространенных операций — например, арифметических — используйте функции из модуля `operator` вместо лямбда-функций:

```python
# YES:
operator.mul(x, y)

# NO:
lambda x, y: x * y
```

### 1.10 Тернарный оператор

Условное выражение (тернарный оператор) — механизм, обеспечивающий более короткий синтаксис для условий.

Хорошо подходит для простых случаев. Каждая часть выражения — условие и оба результата — должна помещаться, как максимум, в одну строку. В более сожных случаях следует испоьзовать обычный `if`.

Вложенные условные выражения не допускаются.

```python
# YES:
one_line = 'yes' if predicate(value) else 'no'

slightly_split = (
    'yes' if predicate(value) else 'no, nein, nyet'
)

the_longest_ternary_style_that_can_be_done = (
    'yes, true, affirmative, confirmed, correct'
    if predicate(value)
    else 'no, false, negative, nay'
)

# NO:
bad_line_breaking = ('yes' if predicate(value) else
                        'no')

portion_too_long = (
    'yes'
    if some_long_module.some_long_predicate_function(
        really_long_variable_name, some_extra_var, and_one_more_var
    )
    else 'no, false, negative, nay')

nested_conditions = (
    'yes'
    if some_condition()
    else 'maybe' if some_another_condition()
    else 'no' if last_condition() else 'break'
)
```

### 1.11 Значения аргументов по умолчанию

Важно учитывать, что значения по умолчанию вычисляются один раз во время инициализации модуля. Это может вызвать проблемы:

- если значением является изменяемый объект, например `list` или `dict`
- если значение задаётся через вызов функции

Не используйте изменяемые объекты в качестве значений по умолчанию в сигнатуре функции или метода.

```python
# YES:
def foo(a, b: Sequence | None = None):
    if b is None:
        b = []

def foo(a, b: Sequence = ()):  # Empty tuple OK since tuples are immutable.
    ...

# NO:
from absl import flags
_FOO = flags.DEFINE_string(...)

def foo(a, b=[]): ...

def foo(a, b: Mapping = {}): ...  # Could still get passed to unchecked code.

def foo(a, b=time.time()): ...  # Is `b` supposed to represent when this module was loaded?

def foo(a, b=_FOO.value): ...  # sys.argv has not yet been parsed...
```

### 1.12 property

Декоратор `property` — дескриптор, позволяющий выполнять вызовов методов для получения и установки атрибута через синтаксис стандартного доступа к атрибуту. Реализация должна соответствовать общим ожиданиям от обычного доступа к атрибутам: она должна быть дешевой, простой и без побочных эффектов. В противном случае следует явно использовать приватный атрибут с геттером и/или сеттером.

### 1.13 True/False evaluations

Python способен оценивать любые значения в логическом контексте. Все «пустые» значения — `0, None, [], {}, ''` — воспринимаются как `False`.

Если возможно, используйте «неявное» значение `True`/`False`. Следует учесть несколько предостережений:

- всегда используйте `if foo is None: ...` (или `is not None`) для проверки `None` значения
- если нужно отделить определение `False` от `None`, используйте конструкцию `if not x and x is not None: ...`
- для последовательностей (строки, списки, кортежи) используйте `if seq: ...` и `if not seq: ...` вместо `if len(seq): ...` и `if not len(seq): ...`.
- при обработке целых чисел неявное приведение может принести больше риска, чем пользы (например, None будет расценен как 0)
- обратите внимание, что "0" (строковый символ) будет восприниматься как `True`
- массивы Numpy могут вызывать исключение в неявном логическом контексте; используйте `.size` атрибут при проверке пустоты np.array (например `if not users.size: ...`)

```python
# YES:
if not users:
    print('no users')

if i % 10 == 0:
    self.handle_multiple_of_ten()

def f(x=None):
    if x is None:
        x = []

# NO:
if len(users) == 0:
    print('no users')

if not i % 10:
    self.handle_multiple_of_ten()

def f(x=None):
    x = x or []
```

### 1.14 Декораторы функций и методов

Декораторы позволяют элегантно модифицировать функцию, исключить часть повторяющегося кода, обеспечить соблюдение инвариантов и т. д.

Декораторы могут выполнять произвольные операции с аргументами функции или возвращаемыми значениями, что приводит к неявному поведению. Кроме того, декораторы выполняются во время инициализации объекта. Для объектов уровня модуля (классов, функций модуля и т. д.) это происходит во время импорта.

Избегайте внешних зависимостей в самом декораторе (например, не полагайтесь на файлы, сокеты, соединения с базой данных и т. д.), поскольку они могут быть недоступны при исполнении декоратора. Декоратор, вызываемый с допустимыми параметрами, должен (насколько это возможно) гарантировать успех во всех случаях.

Никогда не используйте `staticmethod`, если только это не требуется для интеграции с API, определенным в существующей библиотеке. Вместо этого напишите функцию уровня модуля.

Используйте `classmethod` только при написании именованного конструктора или подпрограммы, специфичной для класса, которая изменяет необходимое глобальное состояние, например кэш всего процесса.

### 1.15 Многопоточность (threading)

Никогда не полагайтесь на атомарность встроенных типов.

Хотя встроенные типы данных Python, такие как словари, предоставляют атомарные операции, существуют крайние случаи, когда они не являются атомарными (например, если `__hash__` или `__eq__` реализованы как методы Python), и на их атомарность не следует полагаться. Вам также не следует полагаться на атомарное присвоение переменных (поскольку это, в свою очередь, зависит от словарей).

Используйте структуру `Queue` модуля `queue` как предпочтительный способ передачи данных между потоками. В противном случае используйте модуль `threading` и его примитивы синхронизации. Переменные, информирующие о состоянии, и `threading.Condition` предпочтительнее низкоуровневых блокировок.

### 1.16 Power Features

Python — чрезвычайно гибкий язык, предоставляющий множество необычных функций, таких как пользовательские метаклассы, доступ к байт-коду, компиляция «на лету», динамическое наследование, переоформление объектов, хаки импорта, рефлексия (например, некоторые варианты использования `getattr()`), модификация внутренних компонентов системы, `__del__` методы, реализующие кастомную процедуру очистки и т. д.

Может быть заманчивым использовать эти «крутые» функции, даже если в них нет абсолютной необходимости. Стоит помнить, что они осложняют чтение, понимание и отладку кода. На первый взгляд (автору в процессе написания кода) так не кажется, но при последующей работе без длительного погружения в контекст, такой код оказывается сложнее, чем более длинный, но простой.

Максимально избегайте подобного функционала в своем коде.

Модули стандартной библиотеки и классы, которые внутренне используют такой функционал, можно использовать (например, `abc.ABCMeta`, `dataclasses` и `enum`).

### 1.17 Modern python: `from __future__ import ...`

Возможность включения некоторых более современных функций с помощью `from __future__ import ...` позволяет заранее использовать функции ожидаемых будущих версий Python.

Это обеспечивает более плавное обновление версии интерпретатора, поскольку изменения можно вносить для каждого файла отдельно, заявляя о совместимости и предотвращая регрессии в этих файлах. Такой код более удобен в сопровождении, поскольку с меньшей вероятностью накапливается технический долг, который будет проблематичным во время будущих обновлений. В то же время может быть нарушена обратная совместимость с более старыми версиями.

Для получения дополнительной информации прочтите [Python future statement definitions](https://docs.python.org/3/library/__future__.html).

### 1.18 Аннотации типов

Вы можете аннотировать код Python с помощью подсказок типов в соответствии с [PEP-484](https://peps.python.org/pep-0484/) и проверять типизацию кода во время сборки с помощью инструмента проверки типов, такого как `mypy`.

Аннотации типов могут находиться в исходном коде или в stub-файле `pyi`. По возможности, аннотации должны быть в исходнике. Используйте файлы `pyi` для сторонних модулей.

```python
def func(a: int) -> list[int]: ...
```

Вы также можете объявить тип переменной, используя аналогичный синтаксис [PEP-526](https://peps.python.org/pep-0526/):

```python
a: SomeType = some_func()
```

Аннотации типов улучшают читаемость и удобство сопровождения вашего кода. Использование статических анализаторов позволят обнаружить ошибки на этапе работы с кодом, а не в рантайме.

Аннотация типов настоятельно рекомендуется.

## 2. codestyle

### 2.1 Точки с запятой

Не завершайте строки точкой с запятой и не используйте точку с запятой для размещения двух инструкций в одной строке.

### 2.2 Длина строки

Максимальная длина строки — 80 символов. Исключения из ограничения:

- длинные импорты.
- URL-адреса, имена путей или длинные флаги в комментариях.
- длинные строковые константы уровня модуля, не содержащие пробелов, которые было бы неудобно разбивать на строки, например URL-адреса или имена путей.
- комментарии для линтеров (например: # pylint: disable=invalid-name)

Не используйте обратный бэкслэш для продолжения строки. Вместо этого используйте переносы внутри скобок.

```python
# YES:
foo_bar(
    self, width, height, color='black', design=None, x='foo', emphasis=None
)

if (
    width == 0
    and height == 0
    and color == 'red'
    and emphasis == 'strong'
):

with (
    very_long_first_expression_function() as spam,
    very_long_second_expression_function() as beans,
    third_thing() as eggs,
):
    place_order(eggs, beans, spam, beans)

# NO:
if width == 0 and height == 0 and \
        color == 'red' and emphasis == 'strong':

    with very_long_first_expression_function() as spam, \
        very_long_second_expression_function() as beans, \
        third_thing() as eggs:
    place_order(eggs, beans, spam, beans)
```

Если литеральная строка не помещается в лимит, используйте круглые скобки для неявной конкатенации:

```python
x = (
    'This will build a very long long '
    'long long long long long long string'
)
```

Предпочитайте разрывать строки на максимально возможном синтаксическом уровне. Если вам необходимо дважды разбить строку, оба раза разорвите ее на одном и том же синтаксическом уровне.

```python
# YES:
bridgekeeper.answer(
    name="Arthur", quest=questlib.find(owner="Arthur", perilous=True)
)

answer = (
    a_long_line().of_chained_methods()
    .that_eventually_provides().an_answer()
)

if (
    config is None
    or 'editor.language' not in config
    or config['editor.language'].use_spaces is False
):
    use_tabs()

# NO:
bridgekeeper.answer(name="Arthur", quest=questlib.find(
    owner="Arthur", perilous=True))

answer = a_long_line().of_chained_methods().that_eventually_provides(
    ).an_answer()

if (config is None or 'editor.language' not in config or config[
        'editor.language'].use_spaces is False):
    use_tabs()
```

В комментариях при необходимости помещайте длинные URL-адреса в отдельную строку.

```python
# YES:
# See details at
# http://www.example.com/us/developer/documentation/api/content/v2.0/csv_file_name_extension_full_specification.html

# NO:
# See details at http://www.example.com/us/developer/documentation/api/content/\
# v2.0/csv_file_name_extension_full_specification.html
```

### 2.3 Круглые скобки

Не используйте круглые скобки без необходимости.

Не используйте их в операторах возврата или условных операторах, за исключением случаев использования круглых скобок для переноса строк или для обозначения кортежа.

```python
# YES:
if foo:
    bar()

while x:
    x = bar()

if x and y:
    bar()

if not x:
    bar()

# For a 1 item tuple the ()s are more visually obvious than the comma.
onesie = (foo,)
return foo
return spam, beans
return (spam, beans)
for (x, y) in dict.items(): ...

# NO:
if (x):
    bar()

if not(x):
    bar()

return (foo)
```

### 2.4 Отступы

Отступы в блоках кода составляют 4 пробела.

Никогда не используйте табы.

При переносе строки используйте дополнительный отступ в 4 пробела. Закрывающие скобки (круглые, квадратные или фигурные) при переносе должны располагаться на отдельной строке и иметь такой же отступ, как и строка с соответствующей открывающей скобкой.

```python
# YES:
# 4-space hanging indent; nothing on first line,
# closing parenthesis on a new line.
foo = long_function_name(
    var_one, var_two, var_three, var_four
)
meal = (
    spam, beans
)
foo = long_function_name(
    var_one,
    var_two,
    var_three,
    var_four,
)
meal = (
    spam,
    beans,
)

# 4-space hanging indent in a dictionary.
foo = {
    'long_dictionary_key':
        long_dictionary_value,
    ...
}

# NO:
# Stuff on first line forbidden.
foo = long_function_name(var_one, var_two, var_three,
    var_four)
meal = (spam,
    beans)

# 2-space hanging indent forbidden.
foo = long_function_name(
  var_one, var_two, var_three,
  var_four
)

# No hanging indent in a dictionary.
foo = {
    'long_dictionary_key':
    long_dictionary_value,
    ...
}
```

#### 2.4.1 Запятые в последовательностях элементов

Завершающие запятые в последовательности элементов указываются только при расположении по 1 элементу на строку, а также для кортежей с одним элементом.

```python
# YES:
golomb3 = [0, 1, 3]
golomb3 = [
    0, 1, 3
]
golomb4 = [
    0,
    1,
    4,
    6,
]

# NO:
golomb3 = [0, 1, 3,]
golomb3 = [
    0, 1, 3,
]
golomb4 = [
    0,
    1,
    4,
    6
]
```

### 2.5 Пустые строки

Функции и классы должны отделяться от остального кода двумя пустыми строками.
Методы класса отделяются друг от друга одной пустой строкой. Первый метод класса отделяется от строки объявления класса одной строкой.

Используйте отдельные пустые строки в функциях или методах по своему усмотрению для группировки отдельных блоков кода.

### 2.6 Пробелы

Следуйте стандартным типографским правилам использования пробелов вокруг знаков препинания.

Никаких пробелов внутри скобок.

```python
# YES:
spam(ham[1], {'eggs': 2}, [])

# NO:
spam( ham[ 1 ], { 'eggs': 2 }, [ ] )
```

Никаких пробелов перед запятой или двоеточием. Используйте пробелы после запятой или двоеточия, за исключением конца строки.

```python
# YES:
if x == 4:
    print(x, y)
x, y = y, x

# NO:
if x == 4 :
    print(x , y)
x , y = y , x
```

Никаких пробелов перед открытой скобкой при вызове функции/метода/класса, индексации и т.д..

```python
# YES:
spam(1)
dict_['key'] = list_[index]

# NO:
spam (1)
dict_ ['key'] = list_ [index]
```

Никаких пробелов в конце строк.

Используйте один пробел вокруг операторов присваивания (`=`), сравнения (`==`, `<`, `>`, `!=`, `<=`, `>=`, `in`, `not in`, `is`, `is not`) и логических значений (`and`, `or`, `not`). Используйте здравый смысл при вставке пробелов вокруг арифметических операторов (`+`, `-`, `*`, `/`, `//`, `%`, `**`).

```python
# YES:
x == 1

# NO:
x<1
```

Никогда не используйте пробелы при указании именованных параметров функции, а также при определении значения параметра по умолчанию (здесь есть исключение: при наличии аннотации типа используйте пробелы вокруг оператора присвоения).

```python
# YES:
def complex(real, imag=0.0): return Magic(r=real, i=imag)
def complex(real, imag: float = 0.0): return Magic(r=real, i=imag)

# NO:
def complex(real, imag = 0.0): return Magic(r = real, i = imag)
def complex(real, imag: float=0.0): return Magic(r = real, i = imag)
```

Не используйте пробелы для вертикального выравнивания значений в последовательных строках (применяется к `:`, `#`, `=` и т.д.):

```python
# YES:
foo = 1000  # comment
long_name = 2  # comment that should not be aligned

dictionary = {
    'foo': 1,
    'long_name': 2,
}

# NO:
foo       = 1000  # comment
long_name = 2     # comment that should not be aligned

dictionary = {
    'foo'      : 1,
    'long_name': 2,
}
```

### 2.7 Shebang

Большинству `.py` файлов нет необходимости начинаться со строки `#!` ([PEP-394](https://peps.python.org/pep-0394/)).

### 2.8 Комментарии и строки документации

Обязательно используйте правильный стиль для модулей, функций, строк документации методов и встроенных комментариев.

#### 2.8.1 Строки документации

Python использует строки документации для документирования кода. Строка документации — это строка, которая является первым оператором в пакете, модуле, классе или функции. Эти строки могут быть извлечены автоматически через метод `__doc__` объекта и использованы `pydoc`. Всегда используйте формат трех двойных кавычек `"""` для строк документации ([PEP 257](https://peps.python.org/pep-0257/)). Строка документации должна быть представлена в виде краткого описания функции/метода/класса/модуля, заканчивающегося точкой. При необходимости более подробного описания после этого должна идти пустая строка, за которой следует остальная часть документации, начинающаяся с той же позиции курсора, что и первая кавычка первой строки.

Строки документации, не содержащие никакой новой информации, не должны использоваться.

```python
# NO:
"""Tests for foo.bar."""
```

#### 2.8.2 Модули

Файлы должны начинаться со строки документации, описывающей содержимое и назначение модуля.

```python
"""A one-line summary of the module or program, terminated by a period.

Leave one blank line.  The rest of this docstring should contain an
overall description of the module or program.  Optionally, it may also
contain a brief description of exported classes and functions and/or usage
examples.

Typical usage example:

foo = ClassFoo()
bar = foo.FunctionBar()
"""
```

#### 2.8.3 Функции и методы

В этом разделе «функция» означает метод, функцию, генератор или property.

Строка документации является обязательной для каждой функции, имеющей одно или несколько из следующих свойств:

- является частью публичного API
- нетривиальный размер
- неочевидная логика

Строка документации должна содержать достаточно информации, чтобы иметь возможность корректно использовать функцию без чтения её кода. Строка документации должна описывать синтаксис вызова функции и ее семантику, но, как правило, не детали ее реализации, если только эти детали не относятся к тому, как функция будет использоваться. Например, функция, которая в качестве побочного эффекта изменяет один из своих входных аргументов, должна отметить это в своей строке документации. В противном случае важные детали реализации, не имеющие отношения к вызывающей стороне, лучше выражать в виде комментариев по коду, чем в строке документации функции.

Строка документации может быть описательной или императивной, но стиль должен быть единообразным в пределах файла.

```python
"""Fetches rows from a Bigtable."""

"""Fetch rows from a Bigtable."""
```

Строка документации для `@property` должна использовать тот же стиль, что и строка документации для атрибута или аргумента функции.

```python
# YES:
"""The Bigtable path."""

# NO:
"""Returns the Bigtable path."""
```

Отдельные аспекты функции должны быть задокументированы в специальных разделах, перечисленных ниже. Каждый раздел начинается со строки заголовка, которая заканчивается двоеточием. Все разделы, кроме заголовка, должны иметь дополнительный отступ в четыре пробела. Эти разделы можно опустить в тех случаях, когда название функции и аргументов достаточно информативны, чтобы их можно было точно описать с помощью однострочной строки документации.

##### Args

Перечислить каждый параметр по имени. Описание должно следовать за именем и отделяться двоеточием, за которым следует пробел или новая строка. Если описание слишком длинное и не помещается в лимит по длине, перенесите строку и используйте дополнительный отступ 4 пробела относительно имени параметра. Описание должно включать требуемые типы, если код не содержит аннотации типов. Если функция принимает `*foo` (набор аргументов переменной длины) и/или `**bar` (произвольные именованные аргументы), они должны быть указаны как `*foo` и `**bar`.

##### Returns (и/или Yields для генераторов)

Опишите семантику возвращаемого значения, включая любую информацию о типе, которую не предоставляет аннотация типов. Если функция возвращает только `None`, этот раздел не требуется. Его также можно опустить, если строка документации начинается с "Return", "Returns", "Yield" или "Yields" и этого достаточно для описания возвращаемого значения — например `"""Returns row from Bigtable as a tuple of strings."""`. Если функция использует `yield` (является генератором), в "Yields:" разделе должен быть задокументирован объект, возвращаемый функцией `next()`, а не сам объект-генератор.

##### Raises

Перечислите все исключения, относящиеся к интерфейсу, с последующим описанием. Используйте аналогичное имя исключения + двоеточие + пробел или новую строку и дополнительный отступ (см. "Args"). Не следует документировать исключения, которые возникают, если API, указанный в строке документации, нарушается.

```python
def fetch_smalltable_rows(
    table_handle: smalltable.Table,
    keys: Sequence[bytes | str],
    require_all_keys: bool = False,
) -> Mapping[bytes, tuple[str, ...]]:
    """Fetches rows from a Smalltable.

    Retrieves rows pertaining to the given keys from the Table instance
    represented by table_handle.  String keys will be UTF-8 encoded.

    Args:
        table_handle: An open smalltable.Table instance.
        keys: A sequence of strings representing the key of each table
            row to fetch.  String keys will be UTF-8 encoded.
        require_all_keys: If True only rows with values set for all keys
            will be returned.

    Returns:
        A dict mapping keys to the corresponding table row data
        fetched. Each row is represented as a tuple of strings. For
        example:

        {b'Serak': ('Rigel VII', 'Preparer'),
        b'Zim': ('Irk', 'Invader'),
        b'Lrrr': ('Omicron Persei 8', 'Emperor')}

        Returned keys are always bytes.  If a key from the keys argument is
        missing from the dictionary, then that row was not found in the
        table (and require_all_keys must have been False).

    Raises:
        IOError: An error occurred accessing the smalltable.
    """
```

Допускается вариант с переносом строки:

```python
def fetch_smalltable_rows(
    table_handle: smalltable.Table,
    keys: Sequence[bytes | str],
    require_all_keys: bool = False,
) -> Mapping[bytes, tuple[str, ...]]:
    """Fetches rows from a Smalltable.

    Retrieves rows pertaining to the given keys from the Table instance
    represented by table_handle.  String keys will be UTF-8 encoded.

    Args:
        table_handle:
            An open smalltable.Table instance.
        keys:
            A sequence of strings representing the key of each table row to
            fetch.  String keys will be UTF-8 encoded.
        require_all_keys:
            If True only rows with values set for all keys will be returned.

    Returns:
        ...

    Raises:
        ...
    """
```

#### 2.8.3.1 Переопределенные методы

Методу, который переопределяет метод базового класса, не требуется строка документации, если он явно помечен `@override` (из модуля `typing_extensions` или `typing`), за исключением случаев, когда поведение переопределяющего метода существенно уточняет контракт базового метода или если необходимо предоставить подробную информацию (например, документирование дополнительных побочных эффектов). В этом случае для переопределяющего метода требуется строка документации, содержащая как минимум эти различия.

```python
from typing_extensions import override

class Parent:
    def do_something(self):
        """Parent method, includes docstring."""

# Child class, method annotated with override.
class Child(Parent):
    @override
    def do_something(self):
        pass

# Child class, but without @override decorator, a docstring is required.
class Child(Parent):
    def do_something(self):
        pass

# Docstring is trivial, @override is sufficient to indicate that docs can be
# found in the base class.
class Child(Parent):
    @override
    def do_something(self):
        """See base class."""
```

#### 2.8.4 Классы

Классы должны иметь строку документации под определением класса. Открытые атрибуты, за исключением "property", должны быть задокументированы здесь в `Attributes` разделе и иметь то же форматирование, что и раздел функции `Args`.

```python
class SampleClass:
    """Summary of class here.

    Longer class information...
    Longer class information...

    Attributes:
        likes_spam: A boolean indicating if we like SPAM or not.
        eggs: An integer count of the eggs we have laid.
    """

    def __init__(self, likes_spam: bool = False):
        """Initializes the instance based on spam preference.

        Args:
            likes_spam: Defines if instance exhibits this preference.
        """
        self.likes_spam = likes_spam
        self.eggs = 0

    @property
    def butter_sticks(self) -> int:
        """The number of butter sticks we have."""
```

Все строки документации класса должны начинаться с однострочного описания того, что представляет собой экземпляр класса. Подклассы `Exception` также должны описывать то, что представляет собой исключение, а не контекст, в котором оно может возникнуть. Строка документации класса не должна повторять ненужную информацию.

```python
# YES:
class CheeseShopAddress:
    """The address of a cheese shop.

    ...
    """

class OutOfCheeseError(Exception):
    """No more cheese is available."""

# NO:
class CheeseShopAddress:
    """Class that describes the address of a cheese shop.

    ...
    """

class OutOfCheeseError(Exception):
    """Raised when no more cheese is available."""
```

#### 2.8.5 block и inline комментарии

Основное правило относительно комментариев по коду — комментарии не нужны. При этом комментарии допустимы в исключительных случаях. Обоснованный критерий применения комментариев — сложные и неочевидные части кода. Если что-то потребуется разъяснять на код-ревью, прокомментируйте это прямо в коде.

В сложных случаях может потребоваться несколько строк комментариев перед началом блока описываемого кода.

```python
# We use a weighted dictionary search to find out where i is in
# the array.  We extrapolate position based on the largest num
# in the array and the array size and then do binary search to
# get the exact number.
```

Для более простых случаев достаточно комментария в конце строки. Для улучшения читаемости такой комментарий должен иметь отступ в 2 пробела от кода и затем один пробел после символа "#" перед текстом самого комментария.

```python
if i & (i - 1) == 0:  # True if i is 0 or a power of 2.
```

### 2.9 Строки

Используйте f-строку, `%` оператор или метод `format` для форматирования строк, даже если все параметры являются строками. Одно соединение с помощью `+` допустимо, но не стоит использовать конкатенацию для форматирования.

```python
# YES:
x = f'name: {name}; score: {n}'
x = '%s, %s!' % (imperative, expletive)
x = '{}, {}'.format(first, second)
x = 'name: %s; score: %d' % (name, n)
x = 'name: %(name)s; score: %(score)d' % {'name':name, 'score':n}
x = 'name: {}; score: {}'.format(name, n)
x = a + b

# NO:
x = first + ', ' + second
x = 'name: ' + name + '; score: ' + str(n)
```

Избегайте использования операторов `+` и `+=` для сборки строки в цикле — в некоторых условиях сборка строки с помощью сложения может привести к квадратичному, а не к линейному времени выполнения. Хотя для подобных операций имеются некоторые оптимизации на уровне CPython — это деталь реализации. Условия, при которых применяется оптимизация, нелегко предсказать и они могут измениться. Вместо этого соберите подстроки в список и используйте метод `join` от пустой строки или запишите каждую подстроку в буфер `io.StringIO`. Эти методы имеют амортизированную линейную сложность по времени выполнения.

```python
# YES:
items = ['<table>']
for last_name, first_name in employee_list:
    items.append('<tr><td>%s, %s</td></tr>' % (last_name, first_name))

items.append('</table>')
employee_table = ''.join(items)

# NO:
employee_table = '<table>'
for last_name, first_name in employee_list:
    employee_table += '<tr><td>%s, %s</td></tr>' % (last_name, first_name)
employee_table += '</table>'
```

Будьте последовательны в выборе символа кавычки строки в файле. Выберите `'` или `"` и придерживайтесь этого. Можно использовать другой символ кавычки в строке, чтобы избежать необходимости экранировать символы кавычек в строке с помощью обратной косой черты.

```python
# YES:
Python('Why are you hiding your eyes?')
Gollum("I'm scared of lint errors.")
Narrator('"Good!" thought a happy Python reviewer.')

# NO:
Python("Why are you hiding your eyes?")
Gollum('The lint. It burns. It burns us.')
Gollum("Always the great lint. Watching. Watching.")
```

Предпочтительно использовать `"""` для многострочных строк, а не `'''`. Для строк документации использование двойных кавычек (`"""`) обязательно.

Форматирование многострочных строк может не совпадать с отступами остальной части программы. Если вам нужно избежать лишних пробелов в строке, используйте либо конкатенацию однострочных строк, либо многострочную строку с отступами и удалением начальных пробелов при помощи `textwrap.dedent()`:

```python
# NO:
long_string = """This is pretty ugly.
Don't do this.
"""

# YES:
long_string = """
    This is fine if your use case can accept
    extraneous leading spaces.
"""

long_string = (
    "And this too is fine if you cannot accept\n"
    "extraneous leading spaces."
)

import textwrap

long_string = textwrap.dedent(
    """This is also fine, because textwrap.dedent()
    will collapse common leading spaces in each line."""
)
```

#### 2.9.1 Логирование

Функции логирования, которым передаются не статичные сообщения, всегда вызывайте со строковым литералом с `%`-плейсхолдерами (не f-строкой!) в качестве первого аргумента и дополнительными аргументами для заполнения. Это позволяет не тратить время на форматирование сообщения, вывод которого не предполагается текущим уровнем логирования.

```python
# YES:
import tensorflow as tf

logger = tf.get_logger()
logger.info('TensorFlow Version is: %s', tf.__version__)

import os
from absl import logging

logging.info('Current $PAGER is: %s', os.getenv('PAGER', default=''))

homedir = os.getenv('HOME')
if homedir is None or not os.access(homedir, os.W_OK):
    logging.error('Cannot write to home directory, $HOME=%r', homedir)

# NO:
import os
from absl import logging

logging.info('Current $PAGER is:')
logging.info(os.getenv('PAGER', default=''))

homedir = os.getenv('HOME')
if homedir is None or not os.access(homedir, os.W_OK):
    logging.error(f'Cannot write to home directory, $HOME={homedir!r}')
```

#### 2.9.2 Сообщения об ошибках

Сообщения об ошибках (сообщения, передаваемые с исключением) должны соответствовать трем правилам:

- сообщение должно точно соответствовать фактическому состоянию ошибки
- интерполированные фрагменты всегда должны быть четко идентифицируемы как таковые
- они должны обеспечивать простую автоматическую обработку (например, поиск)

```python
# YES:
if not 0 <= p <= 1:
    raise ValueError(f'Not a probability: {p=}')

try:
    os.rmdir(workdir)
except OSError as error:
    logging.warning(
        'Could not remove directory (reason: %r): %r', error, workdir
    )

# NO:
if p < 0 or p > 1:  # PROBLEM: also false for float('nan')!
    raise ValueError(f'Not a probability: {p=}')

try:
    os.rmdir(workdir)
except OSError:
    # PROBLEM: Message makes an assumption that might not be true:
    # Deletion might have failed for some other reason, misleading
    # whoever has to debug this.
    logging.warning('Directory already was deleted: %s', workdir)

try:
    os.rmdir(workdir)
except OSError:
    # PROBLEM: The message is harder to grep for than necessary, and
    # not universally non-confusing for all possible values of `workdir`.
    # Imagine someone calling a library function with such code
    # using a name such as workdir = 'deleted'. The warning would read:
    # "The deleted directory could not be deleted."
    logging.warning('The %s directory could not be deleted.', workdir)
```

### 2.10 Файлы, сокеты и подобные ресурсы с отслеживанием состояния

Следует явно закрывать файлы и сокеты по завершении работы с ними. Это правило естественным образом распространяется на любые ресурсы, которые внутренне используют сокеты (соединения с базой данных и т.д.), а также на другие ресурсы, которые необходимо закрыть аналогичным образом. Оставление файлов, сокетов или других подобных объектов с состоянием открытыми без необходимости имеет много недостатков:

- они могут потреблять ограниченные системные ресурсы, например файловые дескрипторы; код, работающий с большим количеством таких объектов, может излишне истощить эти ресурсы, если они не будут возвращаться системе сразу после использования
- если файлы будут оставаться открытыми, это может помешать другим действиям, например их перемещению или удалению, а также размонтированию файловой системы
- файлы и сокеты, используемые в программе, могут быть случайно прочитаны или записаны после логического закрытия; если же они действительно закрыты, попытки чтения или записи из них вызовут исключения, что приведет к более раннему обнаружению проблемы.

Более того, хотя файлы и сокеты (а также некоторые ресурсы с аналогичным поведением) автоматически закрываются при удалении объекта, привязка состояния ресурса к времени жизни объекта является плохой практикой:

- нет никаких гарантий относительно того, в какой момент интерпретатор вызовет метод `__del__`. В разных реализациях Python используются разные методы управления памятью, (например — отложенная сборка мусора), которые могут произвольно увеличивать время жизни объекта
- неожиданные ссылки на объект, например, в глобальных переменных или трассировках исключений, могут удерживать его дольше, чем предполагалось

Предпочтительный способ управления файлами и подобными ресурсами — использование менеджера контекста:

```python
with open("hello.txt") as hello_file:
    for line in hello_file:
        print(line)
```

Для файло-подобных объектов, для которых не реализован менеджер контекста, используйте `contextlib.closing()`:

```python
import contextlib

with contextlib.closing(urllib.urlopen("http://www.python.org/")) as front_page:
    for line in front_page:
        print(line)
```

В редких случаях, когда управление ресурсами на основе контекста невозможно, документация по коду должна четко объяснять, как управляется время жизни ресурса.

### 2.11 Комментарии TODO

Используйте `TODO` комментарии для кода, который является временным, краткосрочным решением или достаточно хорошим, но не идеальным.

Комментарий начинается со слова `TODO`, написанного заглавными буквами, последующего двоеточия и ссылки на ресурс, содержащий контекст, в идеале — ссылку на тикет в системе трекинга задач. После этого фрагмента добавьте пояснительную строку, начинающуюся после дефиса. Целью является создание единообразного `TODO` формата, в котором можно будет осуществлять поиск и получать более подробную информацию.

```python
# TODO: crbug.com/192795 — Investigate cpufreq optimizations.
```

Избегайте добавления `TODO`, которые ссылаются на отдельного человека или команду в качестве контекста:

```python
# TODO: @yourusername — File an issue and use a '*' for repetition.
```

Если ваш `TODO` запрос имеет форму «В будущем сделайте что-нибудь», убедитесь, что вы включили либо очень конкретную дату («Исправить к ноябрю 2009 г.»), либо очень конкретное событие («Удалите этот код, когда все клиенты смогут обрабатывать ответы XML»."), которые будущие специалисты по сопровождению кода поймут.

### 2.12 Форматирование импортов

Импорт должен быть выделен в отдельные строки.

```python
# YES:
import os
import sys
from collections.abc import Mapping, Sequence
from typing import Any, NewType

# NO:
import os, sys
```

Импорт всегда помещается в начало файла, сразу после комментариев и строк документации модуля, а также перед глобальными переменными и константами модуля. Импорт следует сгруппировать от наиболее общего до наименее общего:

- `from __future__ import ...`
- импорт модулей и пакетов стандартной библиотеки Python
- импорт сторонних модулей или пакетов
- импорт модулей и пакетов текущего проекта

Группы импортов разделяются одной пустой строкой.

```python
from __future__ import annotations

import sys

import tensorflow as tf

from otherproject.ai import mind
```

Внутри каждой группы импорты должны быть отсортированы лексикографически, без учета регистра, в соответствии с полным путем к пакету каждого модуля.

```python
import collections
import queue
import sys

import bs4
import cryptography
import tensorflow as tf
from absl import app, flags

from book.genres import scifi
from myproject.backend import huxley
from myproject.backend.hgwells import time_machine
from myproject.backend.state_machine import main_loop
from otherproject.ai import body, mind, soul
```

### 2.13 Statements

Обычное правило — одна инструкция в строке.

Однако вы можете поместить результат в ту же строку, что и условие, только если весь оператор умещается в одной строке. В частности, вы никогда не должны делать это для `try - except` (поскольку `try` и `except` не могут помещаться в одной строке).

```python
# YES:
if foo: bar(foo)

# NO:
if foo: bar(foo)
else:   baz(foo)

try:               bar(foo)
except ValueError: baz(foo)

try:
    bar(foo)
except ValueError: baz(foo)
```

### 2.14 Геттеры и сеттеры

Функции получения и установки (также называемые аксессорами и мутаторами) следует использовать, когда они обеспечивают значимую роль или поведение для получения или установки значения переменной.

В частности, их следует использовать, когда получение или установка переменной подразумевает какие-либо дополнительные действия.

Если, например, пара геттер/сеттер просто читает и записывает внутренний атрибут, вместо этого внутренний атрибут следует сделать общедоступным. Для сравнения: если установка значения переменной влечёт за собой инвалидацию какого-либо состояния или пересчёт каких-либо данных, для этого должен использоваться сеттер. Вызов функции намекает на то, что происходит потенциально нетривиальная операция. `property` может быть хорошим вариантом, когда необходима простая логика или при рефакторинге.

Геттеры и сеттеры должны следовать рекомендациям по именованию, таким как `get_foo()` и `set_foo()`.

### 2.15 Именование

- module_name
- package_name
- ClassName
- method_name
- ExceptionName
- function_name
- GLOBAL_CONSTANT_NAME
- global_var_name
- instance_var_name
- function_parameter_name
- local_var_name
- query_proper_noun_for_thing
- send_acronym_via_https

Имена функций, имена переменных и имена файлов должны быть описательными; избегайте сокращений. В частности, не используйте сокращения, которые являются двусмысленными или незнакомыми за пределами вашего проекта, а также не сокращайте, удаляя буквы внутри слова.

Всегда используйте `.py` расширение имени файла. Никогда не используйте тире в имени файлов.

#### 2.15.1 Имена, которых следует избегать

- односимвольные имена, за исключением отдельных особых случаев:
    - счётчики или итераторы (например i, j, k, v и др.)
    - `e` в качестве идентификатора исключения в `try - except`
    - `f` как дескриптор файла в `with`
    - переменные частного типа (например `_T = TypeVar("_T"), _P = ParamSpec("_P")`)

    Будьте внимательны и не злоупотребляйте односимвольными именами. Вообще говоря, описательность должна быть пропорциональна сфере видимости имени. Например, `i` может быть приемлемым именем для пятистрочного блока кода, но в пределах нескольких вложенных областей оно, вероятно, будет слишком расплывчатым.

- тире в любом имени пакета/модуля
- `__double_leading_and_trailing_underscore__` имена (зарезервированы Python)
- оскорбительные термины
- имена, которые без необходимости включают тип переменной (например: id_to_name_dict)

#### 2.15.2 Соглашения об именах

- «внутренний» означает внутренний по отношению к модулю, защищенный или частный внутри класса
- добавление одного подчеркивания `_` в некоторой степени обеспечивает защиту переменных и функций модуля (линтеры будут помечать доступ к защищенному элементу); обратите внимание, что при этом модульные тесты могут получать доступ к защищенным константам из тестируемых модулей
- добавление двойного подчеркивания `__` к переменной экземпляра или методу фактически делает переменную или метод частной для своего класса (за счёт искажения имени); такой подход нежелателен, поскольку влияет на читаемость и тестируемость, не являясь при этом по настоящему конфиденциальным; предпочтительно использовать одно подчеркивание
- поместите связанные классы и функции верхнего уровня вместе в модуль; в отличие от Java, нет необходимости ограничивать себя одним классом для каждого модуля
- используйте `CapWords` для имен классов, но `low_with_under.py` для имен модулей; хотя в кодовой базе существуют некоторые старые модули с именем `CapWords.py`, сейчас это не рекомендуется, потому что это сбивает с толку, когда модуль имеет одинаковое с классом имя («подождите, я указал `import StringIO` или `from StringIO import StringIO`?»)
- новые файлы модульных тестов следуют именам методов `low_with_under`, совместимым с [PEP-8](https://peps.python.org/pep-0008/), например `test_<method_under_test>_<state>.` Для согласованности с устаревшими модулями, в которых применяется `CapWords` в именах методов, подчеркивания используются для разделения логических компонентов имени — `test_<MethodUnderTest>_<state>`.

#### 2.15.3 Именование файлов

Имена файлов Python должны иметь `.py` расширение и не должны содержать дефисы. Если вы хотите, чтобы исполняемый файл был доступен без расширения, используйте символьную ссылку или промежуточный bash-скрипт, содержащий `exec "$0.py" "$@"`.

#### 2.15.4 Рекомендации, основанные на рекомендациях Гвидо

|Тип|Публичный|Приватный|
|---|---------|----------|
|Пакеты|lower_with_under||
|Модули|lower_with_under|_lower_with_under|
|Классы|CapWords|_CapWords|
|Исключения|CapWords||
|Функции|lower_with_under()|_lower_with_under()|
|Глобальные/классовые константы|CAPS_WITH_UNDER|_CAPS_WITH_UNDER|
|Глобальные переменные/классовые переменные|lower_with_under|_lower_with_under|
|Переменные экземпляра|lower_with_under|_lower_with_under|
|Имена методов|lower_with_under()|_lower_with_under()|
|Параметры функции/метода|lower_with_under||
|Локальные переменные|lower_with_under||

#### 2.15.5 Математическая запись

Для математически тяжелого кода предпочтительны короткие имена переменных, если они соответствуют установленным обозначениям в справочной статье или алгоритме, даже если они нарушают руководство по стилю. При этом укажите источник всех соглашений об именах в комментарии или строке документации или, если источник недоступен, четко задокументируйте соглашения об именах.

### 2.16 main

Все модули должны быть импортируемыми. Если файл предназначен для использования в качестве исполняемого файла, его основная функциональность должна заключаться в функции `main()`, а её вызов должен происходить в блоке условия `if __name__ == '__main__'`, чтобы исключить исполнение кода при импорте модуля.

При использовании absl используйте app.run:

```python
from absl import app
...

def main(argv: Sequence[str]):
    # process non-flag arguments
    ...

if __name__ == '__main__':
    app.run(main)
```

В противном случае используйте:

```python
def main():
    ...

if __name__ == '__main__':
    main()
```

### 2.17 Длина функции

Отдавайте предпочтение небольшим и сфокусированным функциям.

Мы понимаем, что длинные функции иногда уместны, поэтому на длину функции не налагается жёсткого ограничения. Если функция превышает примерно 40 строк, подумайте, можно ли ее разбить без ущерба для структуры программы.

Если ваши функции будут короткими и простыми, другим людям будет легче читать и изменять ваш код.

При работе с некоторым кодом вы можете встретить длинные и сложные функции. Не пугайтесь изменять существующий код: если работа с такой функцией вызывает сложности, или возникают трудности с отладкой, или вы хотите использовать часть её функционала в нескольких разных контекстах, рассмотрите возможность разбиения функции на более мелкие и более управляемые части.

### 2.18 Аннотации типов

#### 2.18.1 Общие правила

Ознакомьтесь с [PEP-484](https://peps.python.org/pep-0484/).

Аннотировать `self` или `cls` не нужно; использование `Self` допускается:

```python
from typing import Self

class BaseClass:
    @classmethod
    def create(cls) -> Self:
        ...

    def difference(self, other: Self) -> float:
        ...
```

Не обязательно аннотировать возвращаемое значение `__init__` (где `None` всегда единственный допустимый вариант).

Вам не обязательно аннотировать все функции в модуле
- аннотируйте общедоступные API.
- используйте рассудительность, чтобы найти хороший баланс между безопасностью и ясностью с одной стороны, и гибкостью с другой
- аннотируйте код, который подвержен ошибкам, связанным с типом (по опыту предыдущих ошибок или исходя из сложности кода)
- аннотируйте код, который трудно понять
- аннотируйте код, когда он становится стабильным с точки зрения типов; в зрелом коде в большинстве случаев вы можете аннотировать все функции, не теряя при этом в гибкости

#### 2.18.2 Разрыв строки

Старайтесь следовать существующим правилам отступов.

Часто после аннотирования многие сигнатуры функций потребуется форматировать в стиле «по одному параметру в строке». При переносе делайте дополнительный отступ в 4 пробела в новой строке. При использовании разрывов строк предпочтительнее размещать каждый параметр и тип возвращаемого значения в отдельной строке и выравнивать закрывающую скобку по `def`:

```python
# YES:
def my_method(
    self,
    first_var: int,
    second_var: Foo,
    third_var: Bar | None,
) -> int:
    ...

# NO:
def my_method(self,
              other_arg: MyLongType | None,
              ) -> dict[OtherLongType, MyLongType]:
    ...
```

Если сигнатура укладывается в одну строку — It's okay.

```python
def my_method(self, first_var: int) -> int:
    ...

def my_method(
    self, first_var: int, second_var: Foo, third_var: Bar | None
) -> int:
    ...
```

Старайтесь не разрывать типы. Но иногда они бывают слишком длинные, чтобы помещаться в одну строку. В таких случаях старайтесь, чтобы подтипы не разбивались.

```python
def my_method(
    self,
    first_var: tuple[
        MyLongType1, MyLongType1, MyLongType1, MyLongType1],
    second_var: list[
        dict[MyLongType3, MyLongType4]],
) -> None:
    ...
```

Если имя и тип слишком длинные, рассмотрите возможность использования псевдонима для типа. В крайнем случае — перенесите аннотацию на отдельную строку с дополнительным отступом.

```python
# YES:
def my_function(
    very_long_variable_name:
        long_module_name.LongTypeName,
) -> None:
    ...

# NO:
def my_function(
    long_variable_name: long_module_name.
        LongTypeName,
) -> None:
    ...
```

#### 2.18.3 Предварительные объявления

Если вам нужно использовать имя класса (из того же модуля), которое еще не определено — например, если вам нужно имя класса внутри объявления этого класса или если вы используете класс, который определен позже в коде — либо используйте from `__future__ import annotations`, либо используйте строковое значение для аннотации.

```python
from __future__ import annotations


class MyClass:
    def __init__(self, stack: Sequence[MyClass], item: OtherClass) -> None:


class OtherClass:
    ...

# -------------------------------------------------------------------------

class MyClass:
    def __init__(self, stack: Sequence['MyClass'], item: 'OtherClass') -> None:


class OtherClass:
    ...
```

#### 2.19.4 Значения по-умолчанию

Согласно [PEP-8](https://peps.python.org/pep-0008/), используйте пробелы вокруг оператора присвоения для аргументов, которые имеют как аннотацию типа, так и значение по умолчанию.

```python
# YES:
def func(a: int = 0) -> int:
    ...

# NO:
def func(a:int=0) -> int:
    ...
```

#### 2.19.5 NoneType

Если аргумент может быть `None` — это должно быть объявлено! Вы можете использовать объединения `|` (рекомендуется в новом коде Python 3.10+) или typing.Optional и typing.Union.

Используйте явную аннотацию `X | None` вместо неявной. Более ранние версии [PEP-484](https://peps.python.org/pep-0484/) допускали интерпретацию `a: str = None` как `a: str | None = None`, но теперь это не является допустимым.

```python
# YES:
def modern_or_union(a: str | int | None, b: str | None = None) -> str:
    ...
def union_optional(a: Union[str, int, None], b: Optional[str] = None) -> str:
    ...

# NO:
def nullable_union(a: Union[None, str]) -> str:
    ...
def implicit_optional(a: str = None) -> str:
    ...
```

#### 2.19.6 Псевдонимы типов

Вы можете объявлять псевдонимы сложных типов. Имя псевдонима должно быть `CapWorded`. Если псевдоним используется только в этом модуле, он должен быть `_Private`.

Обратите внимание: `TypeAlias` аннотация поддерживается только в версиях 3.10+.

```python
from typing import TypeAlias

_LossAndGradient: TypeAlias = tuple[tf.Tensor, tf.Tensor]
ComplexTFMap: TypeAlias = Mapping[str, _LossAndGradient]
```

#### 2.19.7 Игнорирование типов

Вы можете отключить проверку типа в строке специальным комментарием # type: ignore.

#### 2.19.8 Типизация переменных

Если внутренняя переменная имеет тип, который трудно или невозможно определить, укажите ее тип с помощью аннотированного присваивания — так же, как это делается с аргументами функции, имеющими значение по умолчанию:

```python
a: Foo = SomeUndecoratedFunction()
```

В старых версиях python (до 3.6) для статического анализа типов использовались комментарии; в новых версиях не следует добавлять комментарии `# type: <type name>`:

```python
a = SomeUndecoratedFunction()  # type: Foo
```

#### 2.19.9 Кортежи и списки

Типизированные списки могут содержать объекты только одного типа. Типизированные кортежи могут иметь либо один повторяющийся тип, либо заданное количество элементов разных типов. Последний обычно используется в качестве типа возвращаемого значения функции.

```python
a: list[int] = [1, 2, 3]
b: tuple[int, ...] = (1, 2, 3)
c: tuple[int, str, float] = (1, "2", 3.5)
```

#### 2.19.10 Переменные типа

В системе типов Python есть дженерики. Переменные типа, такие как `TypeVar` и `ParamSpec`, являются распространенным способом их использования.

Пример:

```python
from collections.abc import Callable
from typing import ParamSpec, TypeVar

_P = ParamSpec("_P")
_T = TypeVar("_T")


def next(l: list[_T]) -> _T:
    return l.pop()


def print_when_called(f: Callable[_P, _T]) -> Callable[_P, _T]:
    def inner(*args: _P.args, **kwargs: _P.kwargs) -> _T:
        print("Function was called")
        return f(*args, **kwargs)
    return inner
```

Для `TypeVar` можно указать ограничения:

```python
AddableType = TypeVar("AddableType", int, float, str)

def add(a: AddableType, b: AddableType) -> AddableType:
    return a + b
```

Распространенной переменной предопределенного типа в модуле typing является `AnyStr`. Используйте его для аннотаций значений, которые могут быть `bytes` или `str`.

```python
from typing import AnyStr


def check_length(x: AnyStr) -> AnyStr:
    if len(x) <= 42:
        return x

    raise ValueError()
```

Переменная типа должна иметь описательное имя, за исключением случаев, когда она:
- недоступна вовне
- не имеет ограничений

```python
# YES:
_T = TypeVar("_T")
_P = ParamSpec("_P")

AddableType = TypeVar("AddableType", int, float, str)
AnyFunction = TypeVar("AnyFunction", bound=Callable)

# NO:
T = TypeVar("T")
P = ParamSpec("P")
_T = TypeVar("_T", int, float, str)
_F = TypeVar("_F", bound=Callable)
```

#### 2.19.11 Типы строк

Не используйте typing.Text в новом коде — только для совместимости с Python 2.

Используйте `str` для строковых/текстовых данных. Для кода, работающего с двоичными данными, используйте `bytes`.

```python
def deals_with_text_data(x: str) -> str:
    ...

def deals_with_binary_data(x: bytes) -> bytes:
    ...
```

Если все строковые типы функции всегда одинаковы, например, если тип возвращаемого значения совпадает с типом аргумента в приведенном выше коде, используйте `AnyStr`.

#### 2.19.12 Импорт аннотаций типов

Для объектов (включая типы, функции и константы) из модулей `typing` или `collections.abc`, используемых для поддержки статического анализа и проверки типов, всегда импортируйте сами объекты. Это делает общие аннотации более краткими.

```python
from collections.abc import Mapping, Sequence
from typing import Any, Generic, cast, TYPE_CHECKING
```

Учитывая, что этот способ импорта добавляет элементы в пространство имён модуля, имена из `typing` или `collections.abc` должны рассматриваться аналогично ключевым словам и не должны быть использованы в вашем коде Python. Если всё же существует конфликт между типом и существующим именем в модуле, импортируйте его с помощью `import x as y`.

```python
from typing import Any as AnyType
```

Предпочитайте использовать встроенные типы в качестве аннотаций, если они доступны. Python поддерживает аннотации типов с использованием параметрических типов контейнеров через [PEP-585](https://peps.python.org/pep-0585/), представленный в Python 3.9.

```python
def generate_foo_scores(foo: set[str]) -> list[float]:
    ...
```

#### 2.19.13 Условный импорт (if <...>: import ...)

Используйте условный импорт только в исключительных случаях, когда во время выполнения необходимо избегать дополнительного импорта, необходимого для проверки типов. Применение такого подхода не рекомендуется.

Импорт, необходимый только для аннотаций типов, можно поместить внутри `if TYPE_CHECKING:` блока. Блок должен располагаться сразу после обычных импортов. В списке импортов не должно быть пустых строк. Отсортируйте этот список так же, как обычные импорты.

```python
import typing

if typing.TYPE_CHECKING:
    import sketch

def f(x: sketch.Sketch): ...
```

#### 2.19.14 Циклические зависимости

Циклические зависимости, вызванные типизацией — признак плохого кода. Такой код является хорошим кандидатом на рефакторинг.

Замените модули, которые создают циклический импорт зависимостей, на `Any`. Установите псевдоним с понятным именем и используйте его атрибуты с именем настоящего типа (любой атрибут `Any is Any`). Определения псевдонимов должны быть отделены от последнего импорта одной строкой.

```python
from typing import Any

some_mod = Any  # some_mod.py imports this module.
    ...

def my_method(self, var: "some_mod.SomeType") -> None:
    ...
```

#### 2.19.15 Дженерики

При аннотировании предпочитайте указывать параметры для универсальных типов; в противном случае параметры будут считаться равными `Any`.

```python
# YES:
def get_names(employee_ids: Sequence[int]) -> Mapping[int, str]:
    ...

# NO: This is interpreted as get_names(employee_ids: Sequence[Any]) -> Mapping[Any, Any]
def get_names(employee_ids: Sequence) -> Mapping:
    ...
```

Если лучшим параметром типа для универсального шаблона является `Any`, укажите его явно; помните, что во многих случаях `TypeVar` может оказаться более подходящим:

```python
# NO:
def get_names(employee_ids: Sequence[Any]) -> Mapping[Any, str]:
    """Returns a mapping from employee ID to employee name for given IDs."""

# YES:
_T = TypeVar('_T')

def get_names(employee_ids: Sequence[_T]) -> Mapping[_T, str]:
    """Returns a mapping from employee ID to employee name for given IDs."""
```

## 3. Напутственные слова

Будьте последовательны.

Если вы редактируете код, потратьте несколько минут на то, чтобы взглянуть на окружающий вас код и определить его стиль. Если они используют `_idx` суффиксы в именах индексных переменных, вам тоже следует это сделать и т.д.

Целью руководства по стилю является организация общего подхода, чтобы люди могли сосредоточиться на том, что вы говорите, а не на том, как вы это говорите. Здесь пердставлены глобальные правила стиля, но локальный стиль также важен. Если код, который вы добавляете в файл, радикально отличается от существующего кода вокруг него, это выбивает читателей из ритма.

Однако существуют пределы последовательности. Согласованность не следует использовать в качестве оправдания для выполнения чего-либо в старом стиле без учета преимуществ нового стиля или тенденции кодовой базы с течением времени сходиться на более новых стилях.
