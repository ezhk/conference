# Ускорение мобильного web
Что значит быстро:
- 0-16мс на отрисовку кадра
- 0-100мс — мгновенный отклик (пользователь этого не видим)
- 100-300мс — заметная задержка — нужны спиннеры
- 300мс-1сек — теряем внимание пользователя
- больше 1-3сек — теряем пользователя

Как измерить:
- lighthouse
- navigation timing API
- performanceObserver
- pageSpeed Insights

Далее будет обзор на примере lighthouse.  
## First meaningfull paint (FMP) — отрисовка полезного на странице
- сокранить TTFB:
  - HTTP ответ может прилетать по частям:
    - непонятно что делать с gzip
      - в node есть стрим, в nginx не дожидаясь всего ответа от proxy_pass: «X-Accel-Buffering: no»
    - статус всегда 200
    - нет HTTP-редиректов
    - не выставить HTTP-only cookie
      - сначала генерируем токен
      - отправляем токен как http-only куку
      - дожидаемся ответа от бека PHPSESSID
      - шифруем PHPSESSID с токеном
      - отправляем его в браузер
  
Добавляем loader:
- плохо добавлять div в head, но это стработает, только поисковый робот может ругаться на невалидную страницу
- можем добавить в head script с тегами body, тогда crawler нормально обработает этот DOM

## Time to interactive — когда пользователь может уже получать отклик от сайта
на 50% сайта TTI 14sec.
Что влияет:
- скорость загрузки скриптов
- из парсинга и выполнения

Скрипты должны быть минимизированы — минимальное количество элементов на старте, которые необходимо «оживлять». Hydrate на react порядка 200ms, несмотря на оптимизации. Есть обсуждение по react, что hydrate только на часть элементов срабатывал, а не на всю страницу.

Rendering:
JS > style > layout > paint > composite.  
Каждый раз как мы работаем с layoutX — это приводит к перевычислению макета,  
поэтому свойства или кешировать или использовать в отложенном режиме (request idle callback).

Как улучшить TTI;
- избегайте reflow/repaint
- уменьшите размер DOM
- сократите количество выполняемого JS

## Большие списки — как с ними работать?
Речь будет про фиды, таблицы, каталог.
Рендерите то, что видит пользователь, остальное скрывайте (не показывать то и скрывать, что не нужно).

Как ускорить:
- использовать virtual scroll
- Visibility observer

Можно показывать пустой контент для некоторых карт (даже если они есть, браузеру надо совсем немного времени на пустые карточки).

Выводы:
- загружайте только то, что нужно
- рендерите то, что видно
- просматривайте вывод ghthouse (CI)
- изучайте вашу аудиторию перед началом оптимизации
- не забывайте следить прогресс браузеров

## Какой impact?
У Гугла есть impact калькулятор, который показывает сколько принесет денег пользователь при уменьшении времени ответа.


# Time vs Quality?
Распределение рабочего времени:
- 1 час на технологии
- 1 час на оптимизацию рабочего процесса
- 6 часов — на создание продукта
Удобнее, когда эти два часа вначале дня.

Разработка кода:
- проверка - автокомпиляция и автообновление результатов:
  - webhack & hotmodule replacement
  - watch / compile daemon — для бекенда (complite daemon следит за изменениями в ФС и выполняет описанные действия)
- мобильные сайты не всегда правильно смотряться на телефоне при разработке на ноутбуке, надо создать инфраструктуру для удобного тестирования
- состояние сайта в URL (достаточно скопировать URL для тестировщика, чтобы понять где проблема)
- разработка через тесирование и разработка через отладчик (отладчик дает очень быструю обратную связь)
  - больше экспериментов
  - больше опыта

Обдумывание:
- ежедневно учиться
- изучать IDE
- изучать отладчик/pretty printer

Программирование:
- навык слепой печати
- повторение про IDE и environment
- работа с git-ом
- разработка через отладчик

Так быстрее цикл: проверил `(check) -->> подумал (think) -->> написал (code) -->> проверил -->> ...  `  

На что не стоит тратить время:
- не имеет смысл делать преждевременные оптимизации и реверс инжиниринг;  
- не стоит иногда бесконечно долго пытаться решить проблему, возможно «свежий взгляд» коллег принесет больший profit.


# ManyChat
[yii](https://www.yiiframework.com)  
[openresty](https://openresty.org)  

Стек: nginx, php, redis, postgres, yii.  
Оптимизации:
- каскадные очереди Redis:
  - каждая очередь для каждого бота
  - общая контрольная очередь, которая содержит информацию о том, содержат ли очереди ботов какие-то данные
- phpfpm -> reactphp, и увеличили количество воркеров reactphp
- сделали процессинг по кластерам (кластер — redis + DB + app, далее «галактики») и reactphp уже обращался напрямую к галактикам
- reactphp перестал справляться (также тормозил деплой) -> nginx+Lua (openresty)


# Маршрутизация в большом приложении на react
В первом приближении начали использовать react router,  
но начали возникать вопросы:
- линки и роуты — строки и нет сложных роутов
- нужно было управлять URL-ами и менять их

Следующим решением был — Router5:
- не зависит от стека технологий (не привязывается к react)

Между react router и router5 нет ничего общего.  
Проект router5 разрабатывается, хотя и не совсем оперативно.  

Из коробки в router5 поддерживается:
- react-router5
- redux-router5

Как подключить router5: HOC / renderprops / hooks.  
Приложение в виде маршрутизации выглядит как дерево: createRouter и передаем дерево в виде списка объектов.  
reactDOM передаем callback-ом в router.start().  
Также есть middleware между переходами роутами:
- canDeactivate
- на улзе перехода action не отрабатыват
- на конечно узле canActivate

Niddleware:
- подгрузка компонентов
- prefetch данных
- ролевая система
- аналитика
- l18n

Для вывода ссылок есть компонента Link с routeParams.  