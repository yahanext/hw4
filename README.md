# Домашнее задание к занятию «Операционные системы. Лекция 2»

### Цель задания

В результате выполнения задания вы:

* познакомитесь со средством сбора метрик node_exporter и средством сбора и визуализации метрик NetData. Такие инструменты позволяют выстроить систему мониторинга сервисов для своевременного выявления проблем в их работе;
* построите простой systemd unit-файл для создания долгоживущих процессов, которые стартуют вместе со стартом системы автоматически;
* проанализируете dmesg, а именно часть лога старта виртуальной машины, чтобы понять, какая полезная информация может там находиться;
* поработаете с unshare и nsenter для понимания, как создать отдельный namespace для процесса (частичная контейнеризация).

### Чеклист готовности к домашнему заданию

1. Убедитесь, что у вас установлен [Netdata](https://github.com/netdata/netdata) c ресурса с предподготовленными [пакетами](https://packagecloud.io/netdata/netdata/install) или `sudo apt install -y netdata`.


### Дополнительные материалы для выполнения задания

1. [Документация](https://www.freedesktop.org/software/systemd/man/systemd.service.html) по systemd unit-файлам.
2. [Документация](https://www.kernel.org/doc/Documentation/sysctl/) по параметрам sysctl.

------

## Задание

1. На лекции вы познакомились с [node_exporter](https://github.com/prometheus/node_exporter/releases). В демонстрации его исполняемый файл запускался в background. Этого достаточно для демо, но не для настоящей production-системы, где процессы должны находиться под внешним управлением. Используя знания из лекции по systemd, создайте самостоятельно простой [unit-файл](https://www.freedesktop.org/software/systemd/man/systemd.service.html) для node_exporter:

    * поместите его в автозагрузку;
    * предусмотрите возможность добавления опций к запускаемому процессу через внешний файл (посмотрите, например, на `systemctl cat cron`);
    * удостоверьтесь, что с помощью systemctl процесс корректно стартует, завершается, а после перезагрузки автоматически поднимается.
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
sudo cp node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin
sudo useradd node_exporter
sudo groupadd node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
sudo touch /etc/systemd/system/node_exporter.service
файл юнита простейший получается такой
vagrant@vagrant:/etc/systemd/system$ cat node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter  

[Install]
WantedBy=multi-user.target
vagrant@vagrant:/etc/systemd/system$ 

далее
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter

vagrant@vagrant:/etc/systemd/system$ sudo systemctl status node_exporter
● node_exporter.service - Node Exporter
     Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2023-03-24 09:33:46 UTC; 2min 50s ago
   Main PID: 2873 (node_exporter)
      Tasks: 6 (limit: 4611)
     Memory: 2.8M
     CGroup: /system.slice/node_exporter.service
             └─2873 /usr/local/bin/node_exporter

Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=node_exporter.go:117 level=info collector=thermal_zone
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=node_exporter.go:117 level=info collector=time
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=node_exporter.go:117 level=info collector=timex
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=node_exporter.go:117 level=info collector=udp_queues
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=node_exporter.go:117 level=info collector=uname
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=node_exporter.go:117 level=info collector=vmstat
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=node_exporter.go:117 level=info collector=xfs
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=node_exporter.go:117 level=info collector=zfs
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.384Z caller=tls_config.go:232 level=info msg="Listening on" address=[::>
Mar 24 09:33:46 vagrant node_exporter[2873]: ts=2023-03-24T09:33:46.385Z caller=tls_config.go:235 level=info msg="TLS is disabled." http2=f>
vagrant@vagrant:/etc/systemd/system$ 

Если необходим конфигурационный файл то изменения в файле 
agrant@vagrant:/etc/systemd/system$ cat node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
EnvironmentFile=/etc/default/node_exporter
ExecStart=/usr/local/bin/node_exporter $OPTIONS 

[Install]
WantedBy=multi-user.target
vagrant@vagrant:/etc/systemd/system$ 

Переменную $OPTIONS определяем в файле /etc/default/node_exporter
```




1. Изучите опции node_exporter и вывод `/metrics` по умолчанию. Приведите несколько опций, которые вы бы выбрали для базового мониторинга хоста по CPU, памяти, диску и сети.

```
node_cpu_seconds_total{cpu="0",mode="iowait"} 
node_cpu_seconds_total{cpu="0",mode="system"} 
node_cpu_seconds_total{cpu="0",mode="user"} 
node_disk_io_time_seconds_total{device="sda"} 6.148
node_disk_io_now{device="sda"} 0
node_filesystem_free_bytes{device="/dev/mapper/ubuntu--vg-ubuntu--lv",fstype="ext4",mountpoint="/"} 2.8220698624e+10
node_memory_MemFree_bytes 3.474272256e+09
node_memory_Buffers_bytes 4.6231552e+07
node_memory_Cached_bytes 3.58531072e+08
node_memory_SwapCached_bytes 0
node_network_receive_packets_total{device="eth0"} 2284
node_network_receive_errs_total{device="eth0"} 0
node_network_transmit_packets_total{device="eth0"} 1642
node_network_transmit_errs_total{device="eth0"} 0

для быстрой оценки состояния хоста должно хватить
```




1. Установите в свою виртуальную машину [Netdata](https://github.com/netdata/netdata). Воспользуйтесь [готовыми пакетами](https://packagecloud.io/netdata/netdata/install) для установки (`sudo apt install -y netdata`). 
   
   После успешной установки:
   
    * в конфигурационном файле `/etc/netdata/netdata.conf` в секции [web] замените значение с localhost на `bind to = 0.0.0.0`;
    * добавьте в Vagrantfile проброс порта Netdata на свой локальный компьютер и сделайте `vagrant reload`:

    ```bash
    config.vm.network "forwarded_port", guest: 19999, host: 19999
    ```

    После успешной перезагрузки в браузере на своём ПК (не в виртуальной машине) вы должны суметь зайти на `localhost:19999`. Ознакомьтесь с метриками, которые по умолчанию собираются Netdata, и с комментариями, которые даны к этим метрикам.
```
Скрин netdata
```
![Скрин netdata](https://github.com/yahanext/hw4/blob/main/scr.png)

1. Можно ли по выводу `dmesg` понять, осознаёт ли ОС, что загружена не на настоящем оборудовании, а на системе виртуализации?
```
При загрузке ос определяет гипервизор и его тип.
###
[    0.000000] DMI: innotek GmbH VirtualBox/VirtualBox, BIOS VirtualBox 12/01/2006
[    0.000000] Hypervisor detected: KVM
###
Но к примеру в vmware можно отредактировать конфигурацию виртуальный машины, что бы скрыть от нее наличие гипервизора.
Если установлены дополнения специфичные для гипервизора, набор модулей будет для каждого гипервизора свой.
```

1. Как настроен sysctl `fs.nr_open` на системе по умолчанию? Определите, что означает этот параметр. Какой другой существующий лимит не позволит достичь такого числа (`ulimit --help`)?
```
fs.nr_open = 1048576 лимит кол-во файловых дескрипторов для процесса,
но по ulimit -n  1024 (максимальное число открытых файлов и в данном варианте такое число не достижимо
```
1. Запустите любой долгоживущий процесс (не `ls`, который отработает мгновенно, а, например, `sleep 1h`) в отдельном неймспейсе процессов; покажите, что ваш процесс работает под PID 1 через `nsenter`. Для простоты работайте в этом задании под root (`sudo -i`). Под обычным пользователем требуются дополнительные опции (`--map-root-user`) и т. д.
```
запускаем screen
root@vagrant:~# unshare -f --pid --mount-proc sleep 1h
далее через ctrl-a c открываем новое окно

root@vagrant:~# ps aux | grep sleep
root        1706  0.0  0.0   5480   580 pts/1    S+   15:43   0:00 unshare -f --pid --mount-proc sleep 1h
root        1707  0.0  0.0   5476   516 pts/1    S+   15:43   0:00 sleep 1h
root        1716  0.0  0.0   6432   656 pts/2    S+   15:43   0:00 grep --color=auto sleep


узнав pid процесса sleep
root@vagrant:~# nsenter --target 1707 --pid --mount
root@vagrant:/# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   5476   516 pts/1    S+   15:43   0:00 sleep 1h
root           2  0.0  0.1   7236  4184 pts/2    S    15:44   0:00 -bash
root          14  0.0  0.0   8888  3320 pts/2    R+   15:45   0:00 ps aux
root@vagrant:/# 
```


1. Найдите информацию о том, что такое `:(){ :|:& };:`. Запустите эту команду в своей виртуальной машине Vagrant с Ubuntu 20.04 (**это важно, поведение в других ОС не проверялось**). Некоторое время всё будет плохо, после чего (спустя минуты) — ОС должна стабилизироваться. Вызов `dmesg` расскажет, какой механизм помог автоматической стабилизации.  
Как настроен этот механизм по умолчанию, и как изменить число процессов, которое можно создать в сессии?

*В качестве решения отправьте ответы на вопросы и опишите, как эти ответы были получены.*

----

### Правила приёма домашнего задания

В личном кабинете отправлена ссылка на .md-файл в вашем репозитории.

-----

### Критерии оценки

Зачёт:

* выполнены все задания;
* ответы даны в развёрнутой форме;
* приложены соответствующие скриншоты и файлы проекта;
* в выполненных заданиях нет противоречий и нарушения логики.

На доработку:

* задание выполнено частично или не выполнено вообще;
* в логике выполнения заданий есть противоречия и существенные недостатки. 
