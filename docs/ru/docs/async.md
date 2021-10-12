# Конкурентность и async / await

Здесь приведена подробная информация об использовании синтаксиса `async def` при написании *функций обработки пути*, а также рассмотрены основы асинхронного программирования, конкурентности и параллелизма.

## Нет времени?

<abbr title="too long; didn't read (основная мысль)"><strong>TL;DR:</strong></abbr>

Допустим, вы используете сторонюю библиотеку, которая требует вызова с ключевым словом `await`:

```Python
results = await some_library()
```

В этом случае *функции обработки пути* необходимо объявлять с использованием синтаксиса `async def`:

```Python hl_lines="2"
@app.get('/')
async def read_results():
    results = await some_library()
    return results
```

!!! note
    `await` можно использовать только внутри функций, объявленных с использованием `async def`.

---

Если вы обращаетесь к сторонней библиотеке, которая с чем-то взаимодействует
(с базой данных, API, файловой системой и т. д.), и не имеет поддержки синтаксиса `await`
(что относится сейчас к большинству библиотек для работы с базами данных), то
объявляйте *функции обработки пути* обычным образом с помощью `def`, например:

```Python hl_lines="2"
@app.get('/')
def results():
    results = some_library()
    return results
```

---

Если вашему приложению (странным образом) не нужно ни с чем взаимодействовать и, соответственно,
ожидать ответа, используйте `async def`.

---

Если вы не уверены, используйте обычный синтаксис `def`.

---

**Примечание**: при необходимости можно смешивать `def` и `async def` в *функциях обработки пути*
и использовать в каждом случае наиболее подходящий синтаксис. А FastAPI сделает с этим всё, что нужно.

В любом из описанных случаев FastAPI работает асинхронно и очень быстро.

Однако придерживаясь указанных советов, можно получить дополнительную оптимизацию производительности.

## Технические подробности

Современные версии Python поддерживают разработку так называемого **"асинхронного кода"** посредством написания **"сопрограмм"** с использованием синтаксиса **`async` и `await`**.

Ниже разберём эту фразу по частям:

* **Асинхронный код**
* **`async` и `await`**
* **Сопрограммы**

## Асинхронный код

Асинхронный код означает, что в языке 💬 есть возможность сообщить машине / программе 🤖,
что в определённой точке кода ей 🤖 нужно будет ожидать завершения выполнения *чего-то ещё* в другом месте. Допустим это *что-то ещё* называется "медленный файл" 📝.

И пока мы ждём завершения работы с "медленным файлом" 📝, компьютер может переключиться для выполнения других задач.

Но при каждой возможности компьютер / программа 🤖 будет возвращаться обратно. Например, если он 🤖 опять окажется в режиме ожидания, или когда закончит всю работу. В этом случае компьютер 🤖 проверяет, не завершена ли какая-нибудь из текущих задач.

Потом он 🤖 берёт первую выполненную задачу (допустим, наш "медленный файл" 📝) и продолжает работу, производя с ней необходимые действия.

Вышеупомянутое "что-то ещё", завершения которого приходится ожидать, обычно относится к достаточно "медленным" операциям <abbr title="Ввода-вывода">I/O</abbr> (по сравнению со скоростью работы процессора и оперативной памяти), например:

* отправка данных от клиента по сети
* получение клиентом данных, отправленных вашей программой по сети
* чтение системой содержимого файла с диска и передача этих данных программе
* запись на диск данных, которые программа передала системе
* обращение к удалённому API
* ожидание завершения операции с базой данных
* получение результатов запроса к базе данных
* и т. д.

Поскольку в основном время тратится на ожидание выполнения операций <abbr title="Ввода-вывода">I/O</abbr>,
их обычно называют операциями, <abbr title="I/O bound">ограниченными скоростью ввода-вывода</abbr>.

Код называют "асинхронным", потому что компьютеру / программе не требуется "синхронизироваться" с медленной задачей и,
будучи в простое, ожидать момента её завершения, с тем чтобы забрать результат и продолжить работу.

Вместо этого в "асинхронной" системе завершённая задача может немного подождать (буквально несколько микросекунд),
пока компьютер / программа занимается другими важными вещами, с тем чтобы потом вернуться,
забрать результаты выполнения и начать их обрабатывать.

"Синхронное" исполнение (в противовес "асинхронному") также называют <abbr title="sequential">"последовательным"</abbr>,
потому что компьютер / программа последовательно выполняет все требуемые шаги перед тем, как перейти к следующей задаче,
даже если в процессе приходится ждать.

### Конкурентность и бургеры

Тот **асинхронный** код, о котором идёт речь выше, иногда называют **"конкурентностью"**. Она отличается от **"параллелизма"**.

Да, **конкурентность** и **параллелизм** подразумевают, что разные вещи происходят примерно в одно время.

Но внутреннее устройство **конкурентности** и **параллелизма** довольно разное.

Чтобы это понять, представьте такую картину:

### Конкурентные бургеры

<!-- The gender neutral cook emoji "🧑‍🍳" does not render well in browsers. In the meantime, I'm using a mix of male "👨‍🍳" and female "👩‍🍳" cooks. -->

Вы идёте со своей возлюбленной 😍 в фастфуд 🍔 и становитесь в очередь, в это время кассир 💁 принимает заказы у посетителей перед вами.

Когда наконец подходит очередь, вы заказываете парочку самых вкусных и навороченных бургеров 🍔, один для своей возлюбленной 😍, а другой себе.

Отдаёте деньги 💸.

Кассир 💁 что-то говорит поварам на кухне 👨‍🍳, теперь они знают, какие бургеры нужно будет приготовить 🍔
(но пока они заняты бургерами предыдущих клиентов).

Кассир 💁 отдаёт вам чек с номером заказа.

В ожидании еды вы идёте со своей возлюбленной 😍 выбрать столик, садитесь и довольно продолжительное время общаетесь 😍
(поскольку ваши бургеры самые навороченные, готовятся они не так быстро ✨🍔✨).

Сидя за столиком с возлюбленной 😍 в ожидании бургеров 🍔, вы отлично проводите время,
восхищаясь её великолепием, красотой и умом ✨😍✨.

Всё ещё ожидая заказ и болтая со своей возлюбленной 😍, время от времени вы проверяете,
какой номер горит над прилавком, и не подошла ли уже ваша очередь.

И вот наконец настаёт этот момент, и вы идёте к стойке, чтобы забрать бургеры 🍔 и вернуться за столик.

Вы со своей возлюбленной 😍 едите бургеры 🍔 и отлично проводите время ✨.

---

А теперь представьте, что в этой небольшой истории вы компьютер / программа 🤖.

В очереди вы просто глазеете по сторонам 😴, ждёте и ничего особо "продуктивного" не делаете.
Но очередь движется довольно быстро, поскольку кассир 💁 только принимает заказы (а не занимается приготовлением еды), так что ничего страшного.

Когда подходит очередь вы наконец предпринимаете "продуктивные" действия 🤓: просматриваете меню, выбираете в нём что-то, узнаёте, что хочет ваша возлюбленная 😍, собираетесь оплатить 💸, смотрите, какую достали карту, проверяете, чтобы с вас списали верную сумму, и что в заказе всё верно и т. д.

И хотя вы всё ещё не получили бургеры 🍔, ваша работа с кассиром 💁 ставится "на паузу" ⏸,
поскольку теперь нужно ждать 🕙, когда заказ приготовят.

Но отойдя с номерком от прилавка, вы садитесь за столик и можете переключить 🔀 внимание
на свою возлюбленную 😍 и "работать" ⏯ 🤓 уже над этим. И вот вы снова очень
"продуктивны" 🤓, мило болтаете вдвоём и всё такое 😍.

В какой-то момент кассир 💁 поместит на табло ваш номер, подразумевая, что бургеры готовы 🍔, но вы не станете подскакивать как умалишённый, лишь только увидев на экране свою очередь. Вы уверены, что ваши бургеры 🍔 никто не утащит, ведь у вас свой номерок, а у других свой.

Поэтому вы подождёте, пока возлюбленная 😍 закончит рассказывать историю (закончите текущую работу ⏯ / задачу в обработке 🤓),
и мило улыбнувшись, скажете, что идёте забирать заказ ⏸.

И вот вы подходите к стойке 🔀, к первоначальной задаче, которая уже завершена ⏯, берёте бургеры 🍔, говорите спасибо и относите заказ за столик. На этом заканчивается этап / задача взаимодействия с кассой ⏹.
В свою очередь порождается задача "поедание бургеров" 🔀 ⏯, но предыдущая ("получение бургеров") завершена ⏹.

### Параллельные бургеры

Теперь представим, что вместо бургерной "Конкурентные бургеры" вы решили сходить в "Параллельные бургеры".

И вот вы идёте со своей возлюбленной 😍 отведать параллельного фастфуда 🍔.

Вы становитесь в очередь пока несколько (пусть будет 8) кассиров, которые по совместительству ещё и повары 👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳, принимают заказы у посетителей перед вами.

При этом клиенты не отходят от стойки и ждут 🕙 получения еды, поскольку каждый
из 8 кассиров идёт на кухню готовить бургеры 🍔, а только потом принимает следующий заказ.

Наконец настаёт ваша очередь, и вы просите два самых навороченных бургера 🍔, один для дамы сердца 😍, а другой себе.

Ни о чём не жалея, расплачиваетесь 💸.

И кассир уходит на кухню 👨‍🍳.

Вам приходится ждать перед стойкой 🕙, чтобы никто по случайности не забрал ваши бургеры 🍔, ведь никаких номерков у вас нет.

Поскольку вы с возлюбленной 😍 хотите получить заказ вовремя 🕙, и следите за тем, чтобы никто не вклинился в очередь,
у вас не получается уделять должного внимание своей даме сердца 😞.

Это "синхронная" работа, вы "синхронизированы" с кассиром/поваром 👨‍🍳. Приходится ждать 🕙 у стойки,
когда кассир/повар 👨‍🍳 закончит делать бургеры 🍔 и вручит вам заказ, иначе его случайно может забрать кто-то другой.

Наконец кассир/повар 👨‍🍳 возвращается с бургерами 🍔 после невыносимо долгого ожидания 🕙 за стойкой.

Вы скорее забираете заказ 🍔 и идёте с возлюбленной 😍 за столик.

Там вы просто едите эти бургеры, и на этом всё 🍔 ⏹.

Вам не особо удалось пообщаться, потому что большую часть времени 🕙 пришлось провести у кассы 😞.

---

В описанном сценарии вы компьютер / программа 🤖 с двумя исполнителями (вы и ваша возлюбленная 😍),
на протяжении долгого времени 🕙 вы оба уделяете всё внимание ⏯ задаче "ждать на кассе".

В этом ресторане быстрого питания 8 исполнителей (кассиров/поваров) 👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳.
Хотя в бургерной конкурентного типа было всего два (один кассир и один повар) 💁 👨‍🍳.

Несмотря на обилие работников, опыт в итоге получился не из лучших 😞.

---

Так бы выглядел аналог истории про бургерную 🍔 в "параллельном" мире.

Вот более реалистичный пример. Представьте себе банк.

До недавних пор в большинстве банков было несколько кассиров 👨‍💼👨‍💼👨‍💼👨‍💼 и длинные очереди 🕙🕙🕙🕙🕙🕙🕙🕙.

Каждый кассир обслуживал одного клиента, потом следующего 👨‍💼⏯.

Нужно было долгое время 🕙 стоять перед окошком вместе со всеми, иначе пропустишь свою очередь.

Сомневаюсь, что у вас бы возникло желание прийти с возлюбленной 😍 в банк 🏦 оплачивать налоги.

### Выводы о бургерах

В нашей истории про поход в фастфуд за бургерами приходится много ждать 🕙,
поэтому имеет смысл организовать конкурентную систему ⏸🔀⏯.

И то же самое с большинством веб-приложений.

Пользователей очень много, но ваш сервер всё равно вынужден ждать 🕙 запросы по их слабому интернет-соединению. 

Потом снова ждать 🕙, пока вернётся ответ.

<!--https://forum.wordreference.com/threads/%D0%9D%D0%BE-%D0%B5%D1%81%D0%BB%D0%B8.3258695/-->
Это ожидание 🕙 измеряется микросекундами, но если всё сложить, то набегает довольно много времени.

Вот почему есть смысл использовать асинхронное ⏸🔀⏯ программирование при построении веб-API.

Большинство популярных фреймворков (включая Flask и Django) создавались
до появления в Python новых возможностей асинхронного программирования. Поэтому
их можно разворачивать с поддержкой параллельного исполнения или асинхронного
программирования старого типа, которое не настолько эффективно.

При том, что основная спецификация асинхронного взаимодействия Python с веб-сервером (ASGI)
была разработана командой Django для внедрения поддержки веб-сокетов.

Именно асинхронность сделала NodeJS таким популярным (несмотря на то, что он не параллельный)
и в этом преимущество Go как языка программирования.

И тот же уровень производительности даёт **FastAPI**.

Поскольку можно использовать преимущества параллелизма и асинхронности вместе,
вы получаете производительность лучше, чем у большинства протестированных NodeJS фреймворков
и на уровне с Go, который является компилируемым языком близким к C <a href="https://www.techempower.com/benchmarks/#section=data-r17&hw=ph&test=query&l=zijmkf-1" class="external-link" target="_blank">(всё благодаря Starlette)</a>.

### Получается, конкурентность лучше параллелизма?

Нет! Мораль истории совсем не в этом.

Конкурентность отличается от параллелизма. Она лучше в **конкретных** случаях, где много времени приходится на ожидание.
Вот почему она зачастую лучше параллелизма при разработке веб-приложений. Но это не значит, что конкурентность лучше в любых сценариях.

Давайте посмотрим с другой стороны, представьте такую картину:

> Вам нужно убраться в большом грязном доме.

*Да, это вся история*.

---

Тут не нужно нигде ждать 🕙, просто есть куча работы в разных частях дома.

Можно организовать очередь как в примере с бургерами, сначала гостиная, потом кухня,
но это ни на что не повлияет, поскольку вы нигде не ждёте 🕙, а просто трёте да моете.

И понадобится одинаковое количество времени с очередью (конкурентностью) и без неё,
и работы будет сделано тоже одинаковое количество.

Однако в случае, если бы вы могли привести 8 бывших кассиров/поваров, а ныне уборщиков 👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳,
и каждый из них (вместе с вами) взялся бы за свой участок дома,
с такой помощью вы бы закончили намного быстрее, делая всю работу **параллельно**.

В описанном сценарии каждый уборщик (включая вас) был бы исполнителем, занятым на своём участке работы.

И поскольку большую часть времени выполнения занимает реальная работа (а не ожидание),
а работу в компьютере делает <abbr title="Центральный процессор (CPU)">ЦП</abbr>,
такие задачи называют <abbr title="CPU bound">ограниченными производительностью процессора</abbr>.

---

Ограничение по процессору проявляется в операциях, где требуется выполнять сложные математические вычисления.

Например:

* Обработка **звука** или **изображений**.
* **Компьютерное зрение**: изображение состоит из миллионов пикселей, в каждом пикселе 3 составляющих цвета,
обработка обычно требует проведения расчётов по всем пикселям сразу.
* **Машинное обучение**: здесь обычно требуется умножение "матриц" и "векторов".
Представьте гигантскую таблицу с числами в Экселе, и все их надо одновременно перемножить.
* **Глубокое обучение**: это область *машинного обучения*, поэтому сюда подходит то же описание.
Просто у вас будет не одна таблица в Экселе, а множество. В ряде случаев используется
специальный процессор для создания и / или использования построенных таким образом моделей.

### Concurrency + Parallelism: Web + Machine Learning

With **FastAPI** you can take the advantage of concurrency that is very common for web development (the same main attractive of NodeJS).

But you can also exploit the benefits of parallelism and multiprocessing (having multiple processes running in parallel) for **CPU bound** workloads like those in Machine Learning systems.

That, plus the simple fact that Python is the main language for **Data Science**, Machine Learning and especially Deep Learning, make FastAPI a very good match for Data Science / Machine Learning web APIs and applications (among many others).

To see how to achieve this parallelism in production see the section about [Deployment](deployment/index.md){.internal-link target=_blank}.

## `async` and `await`

Modern versions of Python have a very intuitive way to define asynchronous code. This makes it look just like normal "sequential" code and do the "awaiting" for you at the right moments.

When there is an operation that will require waiting before giving the results and has support for these new Python features, you can code it like:

```Python
burgers = await get_burgers(2)
```

The key here is the `await`. It tells Python that it has to wait ⏸ for `get_burgers(2)` to finish doing its thing 🕙 before storing the results in `burgers`. With that, Python will know that it can go and do something else 🔀 ⏯ in the meanwhile (like receiving another request).

For `await` to work, it has to be inside a function that supports this asynchronicity. To do that, you just declare it with `async def`:

```Python hl_lines="1"
async def get_burgers(number: int):
    # Do some asynchronous stuff to create the burgers
    return burgers
```

...instead of `def`:

```Python hl_lines="2"
# This is not asynchronous
def get_sequential_burgers(number: int):
    # Do some sequential stuff to create the burgers
    return burgers
```

With `async def`, Python knows that, inside that function, it has to be aware of `await` expressions, and that it can "pause" ⏸ the execution of that function and go do something else 🔀 before coming back.

When you want to call an `async def` function, you have to "await" it. So, this won't work:

```Python
# This won't work, because get_burgers was defined with: async def
burgers = get_burgers(2)
```

---

So, if you are using a library that tells you that you can call it with `await`, you need to create the *path operation functions* that uses it with `async def`, like in:

```Python hl_lines="2-3"
@app.get('/burgers')
async def read_burgers():
    burgers = await get_burgers(2)
    return burgers
```

### More technical details

You might have noticed that `await` can only be used inside of functions defined with `async def`.

But at the same time, functions defined with `async def` have to be "awaited". So, functions with `async def` can only be called inside of functions defined with `async def` too.

So, about the egg and the chicken, how do you call the first `async` function?

If you are working with **FastAPI** you don't have to worry about that, because that "first" function will be your *path operation function*, and FastAPI will know how to do the right thing.

But if you want to use `async` / `await` without FastAPI, <a href="https://docs.python.org/3/library/asyncio-task.html#coroutine" class="external-link" target="_blank">check the official Python docs</a>.

### Other forms of asynchronous code

This style of using `async` and `await` is relatively new in the language.

But it makes working with asynchronous code a lot easier.

This same syntax (or almost identical) was also included recently in modern versions of JavaScript (in Browser and NodeJS).

But before that, handling asynchronous code was quite more complex and difficult.

In previous versions of Python, you could have used threads or <a href="https://www.gevent.org/" class="external-link" target="_blank">Gevent</a>. But the code is way more complex to understand, debug, and think about.

In previous versions of NodeJS / Browser JavaScript, you would have used "callbacks". Which leads to <a href="http://callbackhell.com/" class="external-link" target="_blank">callback hell</a>.

## Coroutines

**Coroutine** is just the very fancy term for the thing returned by an `async def` function. Python knows that it is something like a function that it can start and that it will end at some point, but that it might be paused ⏸ internally too, whenever there is an `await` inside of it.

But all this functionality of using asynchronous code with `async` and `await` is many times summarized as using "coroutines". It is comparable to the main key feature of Go, the "Goroutines".

## Conclusion

Let's see the same phrase from above:

> Modern versions of Python have support for **"asynchronous code"** using something called **"coroutines"**, with **`async` and `await`** syntax.

That should make more sense now. ✨

All that is what powers FastAPI (through Starlette) and what makes it have such an impressive performance.

## Very Technical Details

!!! warning
    You can probably skip this.

    These are very technical details of how **FastAPI** works underneath.

    If you have quite some technical knowledge (co-routines, threads, blocking, etc) and are curious about how FastAPI handles `async def` vs normal `def`, go ahead.

### Path operation functions

When you declare a *path operation function* with normal `def` instead of `async def`, it is run in an external threadpool that is then awaited, instead of being called directly (as it would block the server).

If you are coming from another async framework that does not work in the way described above and you are used to define trivial compute-only *path operation functions* with plain `def` for a tiny performance gain (about 100 nanoseconds), please note that in **FastAPI** the effect would be quite opposite. In these cases, it's better to use `async def` unless your *path operation functions* use code that performs blocking <abbr title="Input/Output: disk reading or writing, network communications.">I/O</abbr>.

Still, in both situations, chances are that **FastAPI** will [still be faster](/#performance){.internal-link target=_blank} than (or at least comparable to) your previous framework.

### Dependencies

The same applies for dependencies. If a dependency is a standard `def` function instead of `async def`, it is run in the external threadpool.

### Sub-dependencies

You can have multiple dependencies and sub-dependencies requiring each other (as parameters of the function definitions), some of them might be created with `async def` and some with normal `def`. It would still work, and the ones created with normal `def` would be called on an external thread (from the threadpool) instead of being "awaited".

### Other utility functions

Any other utility function that you call directly can be created with normal `def` or `async def` and FastAPI won't affect the way you call it.

This is in contrast to the functions that FastAPI calls for you: *path operation functions* and dependencies.

If your utility function is a normal function with `def`, it will be called directly (as you write it in your code), not in a threadpool, if the function is created with `async def` then you should `await` for that function when you call it in your code.

---

Again, these are very technical details that would probably be useful if you came searching for them.

Otherwise, you should be good with the guidelines from the section above: <a href="#in-a-hurry">In a hurry?</a>.
