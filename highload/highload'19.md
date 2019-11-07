# Highload++ 2019

## MySQL

### Group Replication

Keywords:

- group commit как объединение коммитов в одну транзакцию
- semi-sync: [rpl_semi_sync_master_wait_for_slave_count](https://dev.mysql.com/doc/refman/5.7/en/replication-options-master.html#sysvar_rpl_semi_sync_master_wait_for_slave_count)
- GTID
- параллельный слейв: slave_parallel_type
- eXtended COMmunocation: если primary пропадает, то мастер переключается

XCom:

- основан на paxos:
  - выбор лидера (каждый раз, mencius это расширяет: время разбивается на временные слоты, где узлы выбираются по очереди)
  - передали транзацию (всем)
  - большинство серверов подтдвердило
  - принимаем транзакцию
- единая последовательность транзакций в кластере

XCom оптимизации:

- обработка дырок в слотах
- пакетная обработка нескольких транзаций

XCom граничения:

- max 9 узлов
- долгая обработка сообщения: узел удаляется из кластера: можем разбить транзакции на несколько пакетов и можно крутить timeout

Multi primary:

- не рекомендуется под высокой нагрузкой менять таблицы
- продвинутый режим
- нет gap locks, read committed
- нет serializable
- DDL - проблема
- FK - проблема, не могут работать на уровне репликации

group_replication_consistency:

- eventual - не ждать
- BEFORE_ON_PRIMARY_FAILOVER

Восстановление репликации:

- mysqldump
- xtrabackup
- mysql enterprise backup
- CLONE plugin (аналогичен xtrabackup/enterpise, но производится когда серверы выключены, копируются все файлы > копирование модифицированных страниц в память > копирование лога транзакций)

MySQL shell — позволяет простым способом добавлять узлы в том числе с интерактивными вопросами, например, использовать clone для восстановления узлов.  

Выводы:

- уже 3 года, но много возможностей
- скорее для 8.0.17+
- нет поддержки WAN
- шифрование связи и данных
- NoSQL через X-протокол
- **очень аккуратно** в production: некоторые запросы могут сложить репликацию

### XtraDB cluster

- синхронная репликация
  - все изменения должны быть записаны на все узлы, в том числе на самый медленный
- основа — galera
- акцент на автономности
- была сделана до GTID, поэтому не используется напрямую бинарный лог, используются внутренности:
  - события складываются в циклический буффер
  - дополнительных хешей не считается

Обработка DML:

- BEGIN
- ...
- COMMIT
  - выделить write-set
  - получить transaction ID
  - передать write-set
  - подождать сертификации
  - вернуть ОК

**Flow control** — если какой-то узел не успевает, он рассылает это сообщение и все узлы замедляют свое выполнение.  
**ProxySQL** — как балансировщик нагрузки, полностью понимает протокол mysql.  

Роли серверов в mysql_galera_cluster.  
dbdeployer позволяет поднимать InnoDB cluster на нескольких узлах.

| | PXC | InnoDB cluster |
|-|-----|----------------|
| Самовосстановление узлов | + | 8.0.17 | 
| Балансировщик нагрузки | ProxySQL | MySQL Router | 
| Multi-master | + | default: single | 
| API/cmd управления | | mysqlshell | 
| WAN | + | |
| Большие транзакции | 8.0 | |
| Массовость технологии | + | |
| Поддерживается percona | + | + |

## bpfTrace

[PDF slides](https://archive.fosdem.org/2019/schedule/event/using_ebpf_for_linux_performance_analyses/attachments/slides/3316/export/events/attachments/using_ebpf_for_linux_performance_analyses/slides/3316/FOSDEM2019_Using_eBPF_for_Linux_Performance_Analyses.pdf)  
[eBPF latest tools](https://github.com/iovisor)  
[Trace utils doc](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)   

eBPF — extended Berkeley packer filter. Появились расширенные структуры данных.

Включён в Linux с 2014. В современных ядрах поддерживается все лучше.

eBPF — программа в байт коде, которая обрабатывает события. То, что в байт коде — позволяет провести верификацию, что там не циклов и пр.

Порядок:

- пользователь создает программу
- загрузка программы в ядро
- подключение krobes/uprobes/tracepoints и тд
- снятие метрик

bpfTrace — значительно проще raw BFP, bcc (правда уже включает в себя набор тулзов).


## Backup в современной инфраструктуре

> Состояние любого бекапа остается неизвестным до тех пор,  
пока его не попытаются восстановить.  

> Бекап — неизменяемые данные на какой-то момент времени.  

### Bareos

- [vchanger](https://sourceforge.net/projects/vchanger/)
  - восстановление не отменяет бекап
  - полные и инкрементальные задания в одно время
- использовать block device как magazine
  - распределение места
  - read ahead можно точечно менять
- сколько заданий хранить на касете (при восстановлении необходимо читать также другие задачи, так как чтение последовательно)
- параметры, на которые стоит обратить внимание:
  - max (?:volume jobs|concurrent jobs|block size|file size)
  - spool (?:attributes|data)
- разное время хранения бекапов
  - использовать copy job
- много маленьких файлов
  - просто вариант бекапа не подходит, так как изменения могут происходить очень быстро
  - бекап snapshot-ами
- конфигурационные файлы вырабатывать по договоренности
- проверять базу bareos (бекап базы bareos также необходим)
  - bareos-dbcheck: несуществующие/дублирующие записи
  - состояние кассет: failed-задания
- собирать логи/метрики с системы бекапов

## Как выбрать SDN для высоких нагрузок

Opestack Neutron:

- функциональный конструктор
- есть Neutron API и ML2 plugin, который взаимодействует с брокером очереди
- transport: rabbitmq — агенты уже вычитывают данные из rabbitmq
- dataplane: openVSwitch

Агенты не хранят состояния (stateless).  
Full sync на стороне агента срабатывает, когда сервер прислал ошибку на действия агента или rabbitmq вернул ошибку. Но когда сущностей на node много, то fullsync идет часами и всё это время сущностями нельзя управлять. И в логах этой информации нет.  
Центром dataplane является, как правило, openVSwtich со своим агентом, логика управления сети приходится на один агент — это приводит с тысячам правил на агенте, и понять их очень непросто.  

Event-based — событийная модель между агентами и серверов Neuron, но часть агентов могут проигнорировать события или агенты потеряли события, это приводит к «дыркам» в настройках dataplane.  
RabbitMQ — при отказе этого компонента весь SDN неработоспособен, при это rabitmq не обладает бесконечной масштабируемостью — становится bottleneck.  
А больше включенной функциональности Neutron — больше событий.  

После этого были варианты:

- переезд на другой SDN
  - tungsten fabric (openContrail)
    - плюсы:
      - имеет широкую функциональность
      - много документации
      - хорошо интегрируется в «L3 на стойку»
    - минусы:
      - не имеет необходимого функционала
      - огромная кодовая база
      - очень большое количество технологий
      - сложная миграция
  - openDayLight
  - OVN
- переработа Opestack Neutron
  - неподходящая архитектура
  - большая кодовая база
  - очень большой time to market

Собственный SDN позволяет выбрать любую архитектуру, предусмотреть возможность миграции, хотя и написать SDN не так легко.  
Появился **SPRUT** — сейчас работает паралелльно с Neutron в production:

- агенты делают постоянный full sync
- они же строят diff состояний и накладывают его на dataplane
- вместо rabbitmq — HTTP REST API с target-состоянием всех объектов

SDN agent:

- при запуске создает коммутатор и поднимает туннели
- начинает мониторить живость туннелей и передает информацию в SDN controller

SDN controller:

- строит граф сети, где вершины — коммутаторы и ребра — туннели
- по алгоритму Декстры находит маршрут между узлами и для всех коммутаторов по сети генерятся правила с какого порта на какой перебрасывать трафик
- все эти данные затем передаются агентам — напоминает OSPF

Уровень NFV (похож на SDN) — агенты располагаются не на всех узлах, а там, где необходимо. Оперирует сетевыми функциаями, такие как DHCP, Router и пр.  
NFV агенты сразу создают каком-то набор сущностей и дальше они связываются в SDN.  

Дальше в дело вступает nova agent qemu, который взаимодействует с sdn-api и уже строится необходимый маршрут.  

SPRUT:

- микросервисная архитектура
- dataplane = OpenVSwitch
- интеграция с Neutron и Nova
- небольшая кодовая база (написано только то, что нужно — только overlay сеть)
- разработка: меньше года
- текущее состояние = закрытый бета-тест

Выводы:

- начинать выбор SDN с нагрузочного тестирования
- мониторинг обязателен
- написание своего SDN иногда выгодно

## Жизнь без блокировок

Подходы к синхронизации:

- использование блокировок: мьтексы и пр.
  - плюсы:
    - интуитивно понятно
    - хорошо исследованная модель
  - минусы:
    - deadlock
    - создает корреляцию между потоками
- неблокирующие алгоритмы

3 группы неблокирующих алгоритмов:

- abstraction-free
- lock-free, если один из потоков вытесняется планировщиком, то это не должно влиять на другие потоки
- wait-free: определенное количество шагов в алгоритме

Как строить алгоритмы lock-free:  
рассматривать алгоритм, как машину состояний с набором переходов и состояний. Система переходит из одного корректного состояния в другое.  
Делаем предположение: конфликты при работе алгоритма должны быть редкими и при их появлении алгоритм видит это и перевыполняет:

- готовим полную/частичную копию
- меняем копию
- подменяем иходный объект на копию
- если конфликт, то удаляем копию и сначала

То есть алгоритм должен уметь откатить свои действия при возниктовении конфликта. И каждое изменение должно разбиваться на части, чтобы избежать единой большой операции.

Wait-free, типы:

1. для каждого испольнителя копируется состояние меняется и отдается на слияние;
2. перед своим действием исполнитель регистрирует свое действие.

Атомарные операции — interlocked в C#.  
Всегда ли нужна линеаризуемость истории — нет:

- достаточно доказать линеаризуемость не всей истории, а участков
- требование может быть избыточным

Кольцевой FIFO буфер — с чтением и записью отдельными потоками.  
Когдастоит применять:

- код синхронизации значителен
- число потоков велико относительно кол-ва ядер

Не забывая про функциональные и нагрузочные тесты.