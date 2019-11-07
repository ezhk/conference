# MySQL
## Group Replication

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


## XtraDB cluster

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


# bpfTrace
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


# Backup в современной инфраструктуре
> Состояние любого бекапа остается неизвестным до тех пор,  
пока его не попытаются восстановить.  

> Бекап — неизменяемые данные на какой-то момент времени.  

