.. _2trie:

Первоначальный формат словарей (отброшенный)
============================================

.. warning::

    Этот формат словарей в pymorphy2 не используется;
    описание - документация по менее удачной попытке
    организовать словари.

    Первоначальная реализация доступна в одном из первых коммитов.
    Рассматривайте описанное ниже как бесполезный на практике
    "исторический" документ.


В `публикации`_ по mystem был описан способ упаковки словарей с использованием
2 trie для "стемов" и "окончаний". В первом прототипе pymorphy2 был
реализован схожий способ; впоследствии я заменил его на другой.

Этот первоначальный формат словарей в моей реализации обеспечивал
скорость разбора порядка 20-60тыс слов/сек (без предсказателя) при потреблении
памяти 30М (с использованием datrie_), или порядка 2-5 тыс слов/сек
при потреблении памяти 5M (с использованием marisa-trie_).

Идея была в том, что слово просматривается с конца, при этом в первом trie
ищутся возможные варианты разбора для данных окончаний; затем для всех
найденных вариантов окончаний "начала" слов ищутся во втором trie;
в результате возвращаются те варианты, где для "начала" и "конца" есть
общие способы разбора.

Основной "затык" в производительности был в том, что для каждого
слова требовалось искать общие для начала и конца номера парадигм.
Это задача о пересечении 2 множеств, для которой мне не удалось найти
красивого решения. Питоний set использовать было нельзя, т.к. это требовало
очень много памяти.

Лучшее, что получалось - id парадигм хранились в 2 отсортированных
массивах, а их пересечение находилось итерацией по более короткому массиву
и "сужающимся" двоичным поиском по более длинному (параллельная итерация по
обоим массивам на конкретных данных оказывалась всегда медленнее).

В pymorphy2 я в итоге решил использовать другой
:ref:`формат словарей <dictionary>`, т.к.

* другой формат проще;
* алгоритмы работы получаются проще;
* скорость разбора получается больше (порядка 100-200 тыс слов/сек без
  предсказателя) при меньшем потреблении памяти (порядка 15M).

Но при этом первоначальный формат потенциально позволяет
тратить еще меньше памяти; некоторые способы ускорения работы
с ним еще не были опробованы.

Уменьшение размера массивов, как мне кажется - наиболее перспективный тут
способ ускорения. Для уменьшения размеров сравниваемых массивов требуется
уменьшить количество парадигм (например, "вырожденных" с пустым стемом).

.. _публикации: http://download.yandex.ru/company/iseg-las-vegas.pdf
.. _marisa-trie: https://github.com/kmike/marisa-trie


Выделение парадигм
------------------

Изначально в словаре из OpenCorpora нет понятия "парадигмы" слова.
Парадигма - это таблица форм какого-либо слова, образец для склонения
или спряжения.

В pymorphy2 выделенные явным образом парадигмы слов необходимы для того,
чтоб склонять неизвестные слова - т.к. при этом нужны образцы для склонения.

Пример исходной леммы::

    375080
    ЧЕЛОВЕКОЛЮБИВ   100
    ЧЕЛОВЕКОЛЮБИВА  102
    ЧЕЛОВЕКОЛЮБИВО  105
    ЧЕЛОВЕКОЛЮБИВЫ  110

Парадигма (пусть будет номер 12345)::

    ""      100
    "А"     102
    "О"     105
    "Ы"     110

Вся лемма при этом "сворачивается" в "стем" и номер парадигмы::

    "ЧЕЛОВЕКОЛЮБИ" 12345

.. note::

    Для одного "стема" может быть несколько допустимых парадигм.

Прилагательные на ПО-
^^^^^^^^^^^^^^^^^^^^^

В словарях у большинства сравнительных прилагательных есть формы на ПО-::

    375081
    ЧЕЛОВЕКОЛЮБИВЕЕ COMP,Qual V-ej
    ПОЧЕЛОВЕКОЛЮБИВЕЕ       COMP,Qual Cmp2
    ПОЧЕЛОВЕКОЛЮБИВЕЙ       COMP,Qual Cmp2,V-ej

Можно заметить, что в этом случае форма слова определяется не только тем,
как слово заканчивается, но и тем, как слово начинается. Алгоритм с разбиением
на "стем" и "окончание" приведет к тому, что все слово целиком будет считаться
окончанием, а => каждое сравнительное прилагательное породит еще одну
парадигму. Это увеличивает общее количество парадигм в несколько раз и делает
невозможным склонение несловарных сравнительных прилагательных, поэтому
в pymorphy2 парадигма определяется как "окончание", "номер грам. информации"
и "префикс".

Пример парадигмы для "ЧЕЛОВЕКОЛЮБИВ"::

    ""      100     ""
    "А"     102     ""
    "О"     105     ""
    "Ы"     110     ""

Пример парадигмы для "ЧЕЛОВЕКОЛЮБИВЕЕ"::

    ""      555     ""
    ""      556     "ПО"
    ""      557     "ПО"

.. note::

    Сейчас обрабатывается единственный префикс - "ПО". В словарях, похоже,
    нет других префиксов, присущих только отдельным формам слова в пределах
    одной леммы.

Упаковка "стемов"
-----------------

"Стемы" - строки, основы лемм. Для их хранения используется структура данных
trie_ (с использованием библиотеки datrie_), что позволяет снизить
потребление оперативной памяти (т.к. некоторые общие части слов не дублируются)
и повысить скорость работы (т.к. в trie можно некоторые операции - например,
поиск всех префиксов данной строки - можно выполнять значительно быстрее,
чем в хэш-таблице).

.. _trie: http://en.wikipedia.org/wiki/Trie
.. _datrie: https://github.com/kmike/datrie

Ключами в trie являются стемы (перевернутые), значениями - список с номерами
допустимых парадигм.

Упаковка tuple/list/set
-----------------------

Для каждого стема требуется хранить множество id парадигм; обычно это
множества из небольшого числа int-элементов. В питоне накладные расходы на
set() довольно велики::

    >>> import sys
    >>> sys.getsizeof({})
    280

Если для каждого стема создать даже по одному пустому экземпляру set,
это уже займет порядка 80М памяти. Поэтому set() не используется;
сначала я заменил их на tuple с отсортированными элементами. В таких tuple
можно искать пересечения за O(N+M) через однопроходный алгоритм,
аналогичный сортировке слиянием, или за O(N*log(M)) через двоичный поиск.

Но накладные расходы на создание сотен тысяч tuple с числами тоже велики,
поэтому в pymorphy 2 они упаковываются в одномерный массив чисел
(``array.array``).

Пусть у нас есть такая структура::

    (
        (10, 20, 30),       # 0й элемент
        (20, 40),           # 1й элемент
    )

Она упакуется в такой массив::

    array.array([3, 10, 20, 30, 2, 20, 40])

Сначала указывается длина данных, затем идет сами данные, потом опять длина
и опять данные, и т.д. Для доступа везде вместо старых индексов
(0й элемент, 1й элемент) используются новые: 0й элемент, 4й элемент.
Чтоб получить исходные цифры, нужно залезть в массив по новому индексу,
получить длину N, и взять следующие N элементов.

Итоговый формат данных
----------------------

Таблица с грам. информацией
^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    ['tag1', 'tag2', ...]

``tag<N>`` - набор грам. тегов, например ``NOUN,anim,masc sing,nomn``.

Этот массив занимает где-то 0.5M памяти.

Парадигмы
^^^^^^^^^

::

    [
        (
            (suffix1, tag_index1, prefix1),
            (suffix2, tag_index2, prefix2),
            ...
        ),
        (
            ...
    ]


``suffix<N>`` и ``prefix<N>`` - это строки с окончанием и префиксом
(например, ``"ЫЙ"`` и ``""``); ``tag_index<N>`` - индекс в таблице
с грам. информацией.

Парадигмы занимают примерно 7-8M памяти.

.. note::

    tuple в парадигмах сейчас не упакованы в линейные структуры;
    упаковка должна уменьшить потребление памяти примерно на 3M.


Стемы
^^^^^

Стемы хранятся в 2 структурах:

* ``array.array`` с упакованными множествами номеров возможных парадигм
  для данного стема::

       [length0, para_id0, para_id1, ..., length1, para_id0, para_id1, ...]

* и trie с ключами-строками и значениями-индексами в массиве значений::

       datrie.BaseTrie(
           'stem1': index1,
           'stem2': index2,
           ...
       )

"Окончания"
^^^^^^^^^^^

Для каждого "окончания" хранится, в каких парадигмах на каких позициях
оно встречается. Эта информация требуется для быстрого поиска нужного слова
"с конца". Для этого используются 3 структуры:

* ``array.array`` с упакованными множествами номеров возможных парадигм
  для данного окончания::

       [length0, para_id0, para_id1, ..., length1, para_id0, para_id1, ...]

  В отличие от аналогичного множества для стемов, номера парадигм могут
  повторяться в пределах окончания.

* ``array.array`` с упакованными множествами индексов в пределах парадигмы::

       [length0, index0, index1, ..., length1, index0, index1, ...]

  Этот массив работает "вместе" с предыдущим, каждому элементу отсюда
  соответствует элемент оттуда - совместно они предоставляют информацию
  о возможных номерах форм в парадигме для всех окончаний.

* trie с ключами-строками и значениями-индексами::

       datrie.BaseTrie(
           'suff1': index1,
           'suff2': index2,
           ...
       )

  По индексу ``index<N>`` можно из предыдущих двух массивов получить наборы
  форм для данного окончания.

.. note::

    Длины хранятся 2 раза. Может, это можно как-то улучшить?

.. _mystem: http://company.yandex.ru/technologies/mystem/
.. _pymorphy 0.5.6: https://pymorphy.readthedocs.io/en/v0.5.6/index.html
