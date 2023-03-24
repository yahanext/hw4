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





1. Изучите опции node_exporter и вывод `/metrics` по умолчанию. Приведите несколько опций, которые вы бы выбрали для базового мониторинга хоста по CPU, памяти, диску и сети.

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
![Скрин netdata](https://github.com/yahanext/hw4/blob/main/scr.png)

1. Можно ли по выводу `dmesg` понять, осознаёт ли ОС, что загружена не на настоящем оборудовании, а на системе виртуализации?

1. Как настроен sysctl `fs.nr_open` на системе по умолчанию? Определите, что означает этот параметр. Какой другой существующий лимит не позволит достичь такого числа (`ulimit --help`)?

1. Запустите любой долгоживущий процесс (не `ls`, который отработает мгновенно, а, например, `sleep 1h`) в отдельном неймспейсе процессов; покажите, что ваш процесс работает под PID 1 через `nsenter`. Для простоты работайте в этом задании под root (`sudo -i`). Под обычным пользователем требуются дополнительные опции (`--map-root-user`) и т. д.

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
