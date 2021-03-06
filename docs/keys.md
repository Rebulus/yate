Ключи
=====

Ключи это способ оптимизировать обращения к элементам входящего дерева и
кэшировать результаты вычисления шаблонов во время наложения.

(более-менее являются аналогом xsl-ной конструкции `<xsl:key>`)


Оптимизация обращений к дереву
------------------------------

Например, у нас есть такое дерево:

    {
        item: [
            { id: 'first', value: 42 },
            { id: 'second', value: 24 },
            ...
        ]
    }

И мы хотим получать ноду по ее `id`:

    item-5 = .item[ .id == "fifth" ]

Для того, чтобы найти нужный нам элемент, приходится пройтись по **всем** нодам в `.item` и у всех
у них проверить, совпадает ли их `id` с заданным.
Т.е. трудоемкость этой операции линейна от количества элементов в нодесете.
Если мы ищем несколько разных элементов, то каждый раз мы все равно перебираем наш нодесет целиком.

Чтобы прооптимизировать этот поиск можно пройтись по нодесету один раз и составить "словарик" соответствий:

    "first"     ->      .item[0]
    "second"    ->      .item[1]
    ...

В `yate` для этого нужно определить ключ таким образом:

    key item(.item, .id) { . }

Ключевое слово `key`, за ним следует имя ключа. В круглых скобках два выражения.
Первое — нодесет, к которому мы хотим иметь доступ. Второй аргумент — выражение,
которое вычисляется для каждого элемента этого нодесета и служит "ключом" в "словарике".
Помимо этого есть еще и тело ключа внутри `{ ... }`. В нашем случае мы хотим получать именно ту ноду,
которая лежит в "словарике", поэтому тело ключа состоит просто из jpath-а `.`.

После чего можно использовать ключ для доступа к элементам:

    item-5 = item("fifth")


Кэширование результатов выполнения шаблонов
-------------------------------------------

Предположим, у нас есть список писем, у каждого письма может быть одна или несколько меток.
Метка отображается в виде цветного прямоугольника с именем метки в нем, являющегося ссылкой на все письма с этой меткой.
С точки зрения html-я, все метки с одним id одинаковы:

    <div class="message-label" style="background-color: #ff0000">
        <a href="#label/123456789">важные письма</a>
    </div>

Поэтому нет необходимости генерить html для такой метки больше одного раза,
даже если у каждого письма в нашем списке есть эта метка.

Определяем такой ключ:

    key message-label(.labels, .id) {
        <div class="message-label" style="background-color: #{ .color }">
            <a href="#label/{ .id }">{ .title }</a>
        </div>
    }

После чего, мы можем использовать ключ как обычную функцию:

    message-label("123456789")


Ленивость
---------

Ключи вычисляются только тогда, когда это действительно нужно.
Если во время наложения шаблона ключ не был вызван ни разу, то никакого оверхеда от определения этого ключа не появилось.

При первом обращении к ключу будет вычислен нодесет, указанный в первом аргументе ключа.
После чего будет составлен "словарик" — т.е. для каждой ноды из этого нодесета будет вычислен "ключ" (второй аргумент).
Затем будет сделана проверка, выполнялось ли уже тело ключа (`{ ... }`) для переданного "ключа" (в последнем примере это `"123456789"`).
Если нет, то оно будет выполнено и сохранено. И не будет вычисляться повторно.


Кэширование
-----------

Важно понимать, что кэширование работает только во время одного наложения шаблонов.
Т.е. при каждом запуске `yr.run()` все кэши сбрасываются.


Прирост быстродействия
----------------------

Нет никакой гарантии, что использование ключей действительно дает выигрыш в производительности.
Очень много разных нюансов, так что лучше проверять все замерами скорости наложения.

