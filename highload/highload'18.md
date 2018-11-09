## Make passwords great again
Речь пойдет про защиту от оффлайн атак, когда утекла БД и возможет локальный подбор.

У паролей достаточно маленькая энтропия — сложность пароля.
Как люди начали работать с безопасностью паролей: начали использовать однонаправленные функции — хэши, из нельзя «провернуть» назад.

Но хэширование достаточно быстро вычисляются, особенно на GPU. Если энтропия пароля меньше 40 бит, то кластер из 4 видеокарт подберет их за минуту. Есть также радужные таблица — берут пароли, считают от них хеши и дальше только матчинг хеша (то есть не надо каждый раз вычислять хэш-функцию), но эти таблицы большие по размеру. Дальше — есть __соль__, защищают от радужных таблиц, но вычисляются эти хэши очень быстрые.

Как замедлить вычисление хеша:

- иетартивно вычислять хеш: sha256(sha256(sha256(…password)))
- сделали специальные функции, чтобы использовать много памяти (argon2, scrypt), но эти функции будут медленно работать и у нас, а простые пароли всё-равно уязвимы

### Можно ли не нагружать бекенд hash-функциями?
FB вынесли вычисление хеша на сторонний сервис, результат вычисление сохраняется в БД.

+ защита от перебора
- если что-то случится с секретным удаленным ключом на этом стороннем сервисе, то случится беда

### Можно ли сделать лучше, без минусов FB?
Что есть представить пароль в виде числа: passwOrd — 112, 97, 115, 155, …
теперь мы можем выполнять в паролями математические операции.  
  
__Sphinx__ — менеджер паролей: берет мастер пароль и домен, вычисляет от них хэш и превращает это в число. С этим числом мы может проводить различные вычисления.           
Shpinx использует замаскированное число a^^r, он берет private key (b) и вычисляет a^^(r*b), где r — большое рандомное число. Пользователь демаскирует число a^^b, на выходе получается число хеш от пароля и домена.

+ защищает оп перебора
+ пароли независимы друг от друга
+ кража мастер-пароля ничего не дает
+ скорость
    - сервер не знает кто к нему обращает
    - клиент не знает, кто ему отвечает
    - секретный ключ все ещё статический

### Можно ли отказаться от статического ключа?
[Protocol Pythia](https://virgilsecurity.com/pythia):

+ сервис знает, кто вводит пароль
+ сервер может математически доказать, что он — это он

        Backend (public key) отправляет ID пользователя наряду с числом a^^r на сервер, есть есть приватный ключ
        S > B: (UserID, a) ^^ (r*k) + Proof
        B: вычисляет число и кладет его в базу
        где (UserID, a) — билинейная операция

+ не нагружает бекенд
+ защита от перебора
+ сервис может блокировать попытки перебора (UserID)
+ сервис ничего не знает о пароле (blinding)
+ клиент уверен, что ему ответил конкретный сервис (ZKP)
+ бесшовная миграция с существующих решений
+ можно менять секретный ключ, update токен delta = k’ * 1/k и для каждой записи y’ = y * delta
- новая математика (эллиптическая кривая, BLS12-381, ZCASH)
- мало библиотек

### PHE
[Экспериментальный сервис passw0rd.io](https://passw0rd.io/)

Был выпущен сервис PHE (Simple password-hardened encryption services):

+ быстрее Pythia ~ x10
+ nist p-265

Процесс регистрации и проверки пароля проходит по разному:

    Backend > enroll? (данные для регистрации) > Service (private key)
    S > generate sNonce, два числа C0 (y*(sNonce, «0»)) и C1 с его ключом, Proof > B
    B: у него тоже есть ключ, от вычисляет два числа на основании своего пароля и cNonce; затем он складывает эти точки, что прислал сервис и что вычислил он и складывает эти точки в базу

Процесс аутентификации:

- в вычисляет число, высчитывает то, что есть в базе и отправляет на сервис
- сервис проверяет точку и отправляет вторую точку, на сторону клиента

Плюсы PHE:

+ пароль не покидает бекенд
+ нужен приватный ключ на клиенте
+ все основные функции pythia

Минусов не нашли.


## Последние изменения в IO-стеке Linux
Типичная БД: память в RAM (shared mem) и page_cache ядра; также есть WAL, сброшенный из памяти на диск. Всё хорошо, пока помещается в памяти и происходит чтение, когда запись — надо из wal buffer всё скидывать на диск через весь стек.
Также бывает затык в записи write-ahead лога, который уже до предела автоматизирован и тут могут помочь лишь настройки linux.

- shared memory может быть очень большим
- сохранение in-memory страниц на диск — большое IO (wal)
- autovacuum postgres (в его агрессивном поведении) также генерирует IO
- cache refill (подъем данных в кеш из диска)
- worker IO postgres: каждый воркер может генерить IO, если вся shared память «попачкана», а записи ещё не сброшены на диск

IO stack:

            Database memory
              |          |
          Direct IO  Page cache
              |          | (block IO)
                BIO layer
    Request Layer (elevator IO scheduler)
                  |
       Block device interface
                Disks

С ядра 2.6 появляются различные schedulers (до этого был Linus elevator):

- CFQ — универсальный, для каждого процесса своя очередь запросов; в случае с БД, где единый общий процесс, не подходит
- deadline — собираем векторы блоков, чтобы оптимально это всё записать
- noop | none — отключает шедулинг, нет сортировок, мерджа, стало хорошо работать с SSD, где есть параллельная запись (а со сбором вектором получалась воронка)

C 3.13 появился blk-mq, начинался как шедулер: давайте использовать для IO параллелизм, который умеет SSD, с использованием честных очередей, привязанных к ядрам CPU. Завязан на NVMe драйвер Linux.

Старый подход:

        CPUs
        | | |
    elevator queue
          |
         disk

Новый подход:

      CPU1       CPU2       CPU3
       |          |          |
    Elevator Q Elevator Q Elevator Q
          |       |        |
                 DISK

Сейчас под BIO layer > blk-mq (Kyber/BFQ IO schedullers): BFQ — согласно некоторый квоте есть очереди, Kyber оторван от этой математики квот. Из blk-mq всё приходит в NVMe драйвер сразу, без пересчета блоков в цилиндры и пр.

[Linux storage stack diagram](https://www.thomas-krenn.com/en/wiki/Linux_Storage_Stack_Diagram)

Сейчас NVMe 3 версии, драйвер может через себя пропускать на один SSD-блок порядка 32GB/sec; в 5 версии такого ограничения нет.

Последние разработки block layer:

- IO polling: появились быстрые SSD, раньше глубоко в драйвере срабатывал механизм «записи» и мы ожидали обработку прерывания, а SSD делает это быстро, поэтому появился механизм опроса о записи, мы сейчас не ждем блокирующий вызов об успешности записи, как раньше
- new IO schedulers Cyber and BFQ
- IO tagging, ввод от предельного приложения можно затегать и дать определенный приоритет
- direct IO improvements (в prostgres это пока не поддерживается — только для WAL без репликации)


## eBPF для анализа производительности в Linux
eBPF — extended Barley packet filter, пошло всё это с создания файрволов: в 1992 ребята из Berkley написали вирт. машину с быстрой пакетной фильтрацией. Дальше решили,  что это неплохой Фреймворк для event processing-а.

eBPF интергрирован в систему тулзов «perf».
Несколько тезисов:

- Есть байт-код eBPF, который можно загрузить в ядро.
- Программа проверяется на размер, что нет бесконечных случайных циклов и пр.
- Для LLVM clang есть target для eBPF.
- Компиляция и загрузка eBPF программ зависит от ядра, поэтому компилятор при запуске.

[__eBPF tracing tools__](http://www.brendangregg.com/ebpf.html)  
[User space vs Kernel](http://www.brendangregg.com/eBPF/linux_ebpf_internals.png)

Инструментарий:

- трейс: [Linux tracing systems](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)
- сэмплинг, можем посмотреть статус системы, посмотреть CPU performance counters

Linux tracing infrastructure:

- type of kernel interface: kProbe, etc
- eBGP code to prepare data
- perf tools, that collect data

### eBPF: active metrics
[Набор tools](http://www.brendangregg.com/eBPF/tracing_quadrant_jul2018.jpg):

- lovisor/bcc
- lovisor/ply
- perf-tools-unstable

Набор инструментов:

    echo "deb [trusted=yes] https://repo.iovisor.org/apt/bionic bionic main" | sudo tee /etc/apt/sources.list.d/iovisor.list
    apt-get update
    apt-get install bcc-tool
    cd /usr/share/bcc/tools/
    ./tcpretrans

Более высокоуровневый интерфейс — [ply](https://github.com/iovisor/ply)

### eBPF: exporting architecture
[eBPF exporter DOC](https://blog.cloudflare.com/introducing-ebpf_exporter/)  
[eBPF exporter github](https://github.com/cloudflare/ebpf_exporter)

### Future
[Bpftrace](http://www.brendangregg.com/blog/2018-10-08/dtrace-for-linux-2018.html)


## Платформа 4К-стриминга на миллион онлайнов
In format: RTMP, webRTC, OKMP.  
Upload out format: RTMP.

Видео: раз с каким-то интервалом выделяется опорный кадр, а дальше отправляются только diff-ы. Изначально использовали FFMPEG, транспонирование для 64FPS сумела только Nvidia P4.

FFMPEG: 2160p > Decoder >4K> rescaler (1440p, 1080p, etc) > encode (1440p, 1080p, etc), но решили сначала 4К скейлить к 2К и дальше его отправлять на rescaler.  
Чтобы поскейлить:

- запустить несколько decoder-ов
- сделать свой transcoder (что и сделал ОК)

CPU vs GPU:

- GPU выше FPS
- CPU выше throughput

Ffmpeg — NVENC:

- не копируй память
- rescale на GPU
- CUBA свободна

Балансировка на CPU/GPU с учетом веса.  
Общая схема обработки запросов: 
Transform > Segmenter (упаковывает потоки в контейнеры) > Download.

HLS, ffmpeg-dash — возможность динамического изменения качества, подменяется следующий опорный карты. У HLS есть оверхед пакетов для 40Gb/sec — несколько десятков млн PPS.

NGINX при проксировании вносил 500ms delay на выдачу: от wait_time при кеширования proxy_cache — [bug report](https://forum.nginx.org/read.php?21,278731).

Tuning:

- проверить SO_RCVBUF и net.ipv4.tcp_wmem
- congestion control и выбор стартового окна: высокий packet loss очень влияет на клиентов, default сейчас cubic, Google предложил BBR => net.ipv4.tcp_congestion_control = bbr, но jitter (отклонение задержек RTT) по-прежнему есть
- использование QUIC

Деградация сервиса:

- throttler
- убирать качество из плеера (но плеер должен уметь переизбирать манифест)


## DNS в Facebook
[contact](linkedin/leoleovich)

Seoul > Oregon: RTT = 75ms.  
Simple HTTP:

    > S
    < S-A
    > A
    > ClientHello
    < Server Hello
    > ChangeCipherSpec
    < ChangeCipherSpec
    > GET
    < Content

Итого 600ms на простой запрос.  
PoP терминируют TCP/TLS максимально близко к пользователем.   

Как устроен PoP внутри:

- L4LB
- L7LB
- здесь нет backend и пользовательский запрос отправляется на backend в основном ДЦ

DNS LB Cartographer и выбор PoP:

- близость к пользователю (по сети)
- capacity (один PoP не может обслужить всех пользователей в регионе)
- health (доступность PoP)
- если недостаточно данный, то использование физической удаленности — Geo Data

Sonar — периодический запрос отправляется не на самый ближайший PoP, чтобы получить общую карту доступности. Всё это поступает в Cartographer, который каждую минуту перестраивает карту доступности и наиболее оптимальных PoP. Cartographer генерирует вывод в tiny DNS и доставляют их до DNS-серверов.

Ситуация становится хуже с публичными DNS-серверами, который может быть где-нибудь на другом континенте и тогда будет выбран PoP максимально близкий в DNS-серверу пользователя.

Internal DNS infra:

- максимально близко к сервисам в DNS
- DDoS на внешнюю инфраструктуру никак не должен сказываться на внутренней

Unbound в качестве DNS-cache и всё это на TinyDNS.  
Также используют ExaBGP для анонсирования IP-адресов:  
```announce route <IP> next-hop <BGP device>```

Python framework создает уникальные таски, который может импортировать данные из любой системы, в том числе DNS.  
DNS Mutator — агрегирует и валидирует данные, если они не прошли проверку, то срабатывает ошибка, если данные валидны, то они попадают в zookeeper, откуда они попадают на presenter, там склеиваются и на выходе попадают в TinyDNS формате и с помощью торрентов попадают на DNS серверы.  
Общая схема выглядит как:

    Servers > DNS Mutator > ZooKeeper > Presenter > Publisher >via torrent> DNS Servers

CDB как хранилище данных — неизменяемая и очень быстрая БД, которая появляется их текстового файла в определенной формате, eg. ```+fqdn:ip:ttl:timestamp:lo``` (lo — client location).

Надежность: на каждом сервере FB ресолвятся рандомные записи.

Dogfooding: если бы мы производили собачий корм, то были бы настолько уверены в его качестве, что если бы сами.  
При прохождении запросов с Unbound в TinyDNS мы тоже не знаем откуда пришел запрос — от пользователя или из корпоративного DNS (в итоге раньше разделяли всё инфраструктуру внутреннего и внешнего DNS). Не так давно появился EDNS — клиент или рекурсивный сервер добавляют клиентскую подсеть в DNS пакет (like a X-Forwarded-For в HTTP). Теперь Unbound инжектирует EDNS и только потом отправляет запрос в TinyDNS.


## Решение проблем высоконагруженной балансировки
[Thrift-pool](https://github.com/qiwi/thrift-pool)

ПИД регулятор — следим за временами ответа нод и при отклонениях времен ответов изменяем вес ноды:

- пропорциональный
- интегрирующий
- дифференцирующий

Большое количество автоколебаний для ПИД регулятора.  

Ещё одна проблема балансировки — слишком большие пулы, когда очередь на входе накапливается на балансировщике, пока разбирается эта очередь — timeout на стороне клиента. Из-за этой очереди qiwi прилег на несколько часов, application не успевал обрабатывать накапливаемую очередь.  
Тут можно уменьшить очередь, таким образом отбросив часть клиентов — отталкиваемся от обычных значений + 20% на всякие пики.

Consistent hashing — как продолжение балансировки по ключу, в этом случае добавляют ноды много раз в кольцо хэшей, чтобы при выходе одной из ном была более равномерная балансировка.


## Replicated service mesh
Default scheme:

    Ingress (LB) or global ingress (LBs) > FrontEnds > Mixer (going to Cache or Egress) > BackEnd > DB

Service reliability goals:

- satisfy all demand:
    - capacity planning/provisioning
    - making use of capacity
- serve within error/latency budget (SLO)
    - prevent/mitigate error/timeouts

In-DC and x-dc LB:

- why and when
    - key to scalability
    - equalize constrained resource utilization
    - keep latency low
- prefer L7 LB
    - l7 fined load balancer granularity (HTTP/RPC req vs connection) and content basing routing
    - l4 cheaper and appropriate for streams

- waterfall by region (RTT based)
- waterfall by order (prio/order based)
- enables:
    - location failover
    - error averse assignments
    - capacity sharing
- prefer auto calculation and adjustment
- isolation trade-off: blast-radius containment (safety) vs resource overprovisioning (money)

Traffic blackholes solutions:

- health checks / readiness probes
- errors averse load assignment

Overload preventative measures:

- use flow control in existing framework (gRPC)
- change canarying
- careful change management: gradual changes rollout
- know your system resources and behavior under overload
- control client retry behavior
- user server and client side throttling in work conserving mode
- use horizontal and vert autoscaling
- know autoscaling reaction speed
- have dedicated hospital nodes

Best practices with system changes:

- config is code, treat as such
- package configs, binary, shared libs hermetically together
- use canary deployment

Caches

- hides requests volume
- wort great on static content
- introduces data inconsistency


Key take-aways:

- ensure x-stack observebility
- user l7 LB
- use autoscaling (V/H, pods/clusters)
- use externally consistent DBs


## Кластер Kubernetes в твоём ноутбуке. Знакомство с minikube
[kubeshop](https://github.com/viteksafronov/kubeshop)

$ kubectl config get-contexts

    CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
    *         minikube   minikube   minikube

$ kubectl get pods

    No resources found.

$ kubectl get pods -n kube-system

    NAME                                    READY   STATUS    RESTARTS   AGE
    coredns-c4cffd6dc-zhcmg                 1/1     Running   1          36m
    etcd-minikube                           1/1     Running   0          8m
    kube-addon-manager-minikube             1/1     Running   1          36m
    kube-apiserver-minikube                 1/1     Running   0          8m
    kube-controller-manager-minikube        1/1     Running   0          8m
    kube-dns-86f4d74b45-fxgs6               3/3     Running   4          36m
    kube-proxy-pgvzk                        1/1     Running   0          7m
    kube-scheduler-minikube                 1/1     Running   0          8m
    kubernetes-dashboard-6f4cfc5d87-98264   1/1     Running   3          36m
    storage-provisioner                     1/1     Running   3          36m


## From Ansible to SaltStack - как и зачем
[Попробовать salt у себя](https://github.com/saltstack/salt-bootstrap)  

Проблемы ansible:

- масштабируемость
- поддержка актуальности
- версионирование
- разделение прав
- аутентификация

Salt:

- pull-модель из коробки (ansible только push)
- REST-api
- LDAP, разделение прав
- деплортивный подход — __что__ хотим получить
- императивный — __как__ хотим получить

На хостах устанавливается minion и подключается на порт 4505 мастера, результат определяет на порт 4506.  
Salt master имеет pillars — параметры раскатки на миньонах, миньон собирает список статичной информации о себе и отправляет их на мастер — grains.

State apply — процесс синхронизации состояния на миньонах со стороны мастера.

Ansible:

- нет агентов
- источник информации — управляющая машина
- inventory и playbook-м
- roles
- в качество основного протокола — ssd

Salt:

- настройка через агенты и их особенности:
    + pull-модель
    + информация о хоста
    + расширяемость
    - выше порог входа
    - сложно разрабатывать
- есть сервер, откуда берется информация
- pillar и top.sls
- states (скрипты, походы на роли ansible)
- зашифрованный протокол

С использованием salt можно посмотреть выполняемые задачи и исключена возможность запуска одновременных задач как в ansible. Но в последнем тоже есть pull-модель: ansible-pull, ansible-tower (non free).

Авторизация salt-api с использованием external_auth модуля. Но модуль LDAP не позволяет авторизацию с multiple domains (базовый домен один, последний в списке) и плохо работает с кириллицей.  
Ansible использует with_items вместо циклов и when в условиях, у salt несколько беднее — меньше фильтров, в последнем в качестве шаблонов — jinja.  
Salt раскатывает быстрее ansible.

Ansible оперирует fail_hard — ошибка в одной части шаблона = остановке всего шаблона, у salt fail_fast по умолчанию — выполнение до конца при падении части таски (можно установить при этом fail_hard на конкретный критичный таск).  
В ansible менее удобны проверки условия шагов — в failed_when надо сравнивать значение с переменной, в которую нужно сначала что-то записать, у salt всё удобнее на примере cmd.run — stateful.

Последовательность запуска:

- require
- order
- watch

Salt-API — веб-обертка над командой salt, которая выполняется на мастере.  
Что делают с его помощью:

- авторизация пользователей (POST /login)
- получение состояния миньонов
- выполнение команды на миньоне (POST /run)

Почему не везде salt:

- кастомная конфигурация инфраструктуры сервиса
- deploy в marathon
- deploy инфраструктуры разработки
- проблема курицы и яйца: чем катать Salt?


Итог:

- salt идеален для раскатки базовой инфраструктуры
- для CI/CD лучше ansible
- salt не смог «убить» ansible
