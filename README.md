# Astrologic - автоматическая оптимизация кода на уровне AST


Пакет Astrologic содержит ряд инструментов, позволяющих "в рантайме" оптимизировать работу ваших функций или даже конструировать новые функции. Все это достигается за счет парсинга и преобразования [AST](https://docs.python.org/3/library/ast.html) исходной функции. Парсинг происходит полностью "под капотом" и невидим для пользователя.


## Оглавление

- [**Дисклеймер**](#дисклеймер)
- [**Установка**](#установка)
- [**Включаем и отключаем блоки кода**](#включаем-и-отключаем-блоки-кода)
- [**Оптимизируем хвостовую рекурсию**](#оптимизируем-хвостовую-рекурсию)


## Дисклеймер

Все, что вы найдете в данном пакете - это демонстрация концепций. Существует огромное количество весьма разумных причин, почему код из данной библиотеки, или из других, содержащих подобные хаки, не должен использоваться в "реальности". Как минимум, манипуляции с AST приводят к тому, что у вас начинает реально исполняться совершенно не тот код, который вы видите в файлах проекта. Средства интроспекции Python тоже совершенно не ожидают таких усовершенствований и могут давать ложную информацию.

Кроме того, большинство предложенных здесь оптимизаций фактически бесполезны. Они оптимизируют вещи, которые, как правило, отъедают совсем немного машинного времени, однако это происходит за счет значительного увеличения времени инициализации объектов в модулях программы.

Если уж вы по собственной глупости решите практически использовать данную библиотеку, пожалуйста, хорошо тестируйте свой код. Иногда вы можете столкнуться с совершенно неожиданным его поведением. Учитывайте, что предложенные здесь функции плохо протестированы (ввиду чисто демонстрационных целей их написания), и содержащиеся в них потенциальные баги могут удивить даже автора.


## Установка

Установите Astrologic через [pip](https://pypi.org/project/astrologic/):

```
$ pip install astrologic
```

## Включаем и отключаем блоки кода

Простейшим из декораторов библиотеки Astrologic является ```@switcher```. Вот пример его использования:

```python
from astrologic import switcher


@switcher(a=False, b=True)
def function():
  print('begin')
  if a:
    print('block a')
  if b:
    print('block b')
  print('end')

function() # Что будет выведено? Проверьте сами!
```

```@switcher``` проходится по унарным условиям (условиям, в которых только один операнд). Если название переменной, которая фигурирует в данном условии, присутствует в именованных аргументах самого декоратора, он делает одну из двух вещей. Если именованный аргумент равен True, он достает данный блок кода из условия заменяет им само условие. Если аргумент равен False, он удаляет условие вместе с блоком кода. То есть функция из примера выше превращается в:

```python
# Тот код, который будет исполнен на машине. Блок кода "if a:" полностью вырезан, а блок "if b:" вытащен из проверки, в то время как сама проверка тоже вырезана.
def function():
  print('begin')
  print('block b')
  print('end')
```

Использовать данный декоратор вы можете, к примеру, чтобы избежать каких-то проверок в вашем коде, основанных на константах. При инициализации функций в начале работы интерпретатора вы указываете, какие блоки кода вам нужны. По сути, if'ы в данном случае служат не по прямому назначению, а работают разметкой кода.

Поскольку ```@switcher``` редактирует функции на уровне AST, в некоторых случаях применение декоратора может ускорить ваш код, полностью убрав ненужные операции из реально исполняемого машиной кода.

У ```@switcher``` есть ряд ограничений и правил использования, которые необходимо учитывать:

- Декоратор ```@switcher``` должен быть первым в списке декораторов, все остальные можно накладывать только поверх. Если не соблюсти это правило, декораторы, лежащие под ```@switcher```, могут просто исчезнуть, будто их не было.
- Никаких else или else if у блоков if, которыми вы управляете через декоратор! У прочих блоков if использовать else можно, поскольку они никаких не модифицируются.

## Оптимизируем хвостовую рекурсию

Еще один крутой трюк с AST - автоматическая оптимизация [хвостовой рекурсии](https://ru.wikipedia.org/wiki/%D0%A5%D0%B2%D0%BE%D1%81%D1%82%D0%BE%D0%B2%D0%B0%D1%8F_%D1%80%D0%B5%D0%BA%D1%83%D1%80%D1%81%D0%B8%D1%8F). Для этого нам понадобится декоратор ```@no_recursion```:


```python
from astrologic import no_recursion

counter = 0

@no_recursion
def recursion():
    global counter
    counter += 1
    if counter != 10000000:
        return recursion()
    return counter

print(recursion()) # Попробуйте выполнить это сами!
```

Как вы, вероятно, знаете, максимальная глубина рекурсии в Python ограничена, и обычно составляет около 1000. Однако код из этого примера отработает и вернет корректный результат, поскольку декоратор ```@no_recursion``` автоматически преобразовал рекурсивный код в итеративный. В отличие от других [решений](https://github.com/0scarB/tail-recursive) данной задачи, здесь не потребовалась менять исходный код функции, кроме как навешиванием на нее декоратора.

Надо сказать, против автоматической оптимизации хвостовой рекурсии [высказывался](http://neopythonic.blogspot.com/2009/04/tail-recursion-elimination.html) даже сам [Гвидо Ван Россум](https://en.wikipedia.org/wiki/Guido_van_Rossum). Его доводы довольно весомы, но они относятся, прежде всего, к внедрению данной возможности непосредственно в интерпретатор. Сделать такую оптимизацию бесшовной, с учетом интерпретируемой и гибкой природы Python, действительно невозможно. Однако это не мешает нам сделать это, пойдя на некоторые ограничения.


<details>
<summary>Вот они:</summary>

- Поддерживается только обычная хвостовая рекурсия. Она выглядит вот так:

```python
def function():
  return function()
```

  Функция вызывает сама себя в блоке return и никак иначе. Никакие другие кейсы, включая, скажем, перекрестную рекурсию (это когда функция А вызывает функцию Б, а та, в свою очередь - функцию А), не покрываются.

- За рекурсию принимается вызов объекта с тем же именем, какое название у функции. Вызов того же имени от другого объекта будет принят за рекурсию!

```python
def function():
  return obj.function() # Декоратор примет это за рекурсию, хотя, очевидно, obj.function != function.
```

- Внутри функции нельзя использовать имена ```is_recursion``` и ```superfunction```. Особенности реализации. Остальные имена можно.

- При использовании ключевого слова "global" вас могут поджидать некоторые неприятности.

- Стек-трейсы, как и предупреждал Гвидо, могут не совсем корректно отражать реальность.

- Как ```@switcher```, ```@no_recursion``` должен быть первым в списке декораторов.

</details>


## Инлайним функции

Следующая оптимизация - инлайновые функции. В данном пакете это работает немного иначе, чем вы, вероятно, ожидаете, если знаете про [такую возможность](https://en.cppreference.com/w/c/language/inline) в C++. Здесь метка ```inline``` вешается не на ту функцию, которую мы инлайним, а на ту, __в__ которую.

```python
from astrologic import inline


def a(c):
    d = c
    print(d)

@inline('a')
def b():
    print('lol')
    c = 'kek'
    a(c)

b() # Попробуйте выполнить это сами!
```
