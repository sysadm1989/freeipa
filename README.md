# Отказоустойчивый FreeIPA

Роль `site.yml` для уже установленных реплик. Первую установку делает
`install.yml`: `ipa-server-install` на одном хосте, затем `ipa-replica-install`
на остальных, и сразу то же укрепление. Пароли в git не попадают.

## Как не ронять домен

Нужны три реплики в разных доменах отказа: не одна стойка и не один гипервизор.
Для небольшой инсталляции на каждой реплике включены CA и DNS. Две реплики с CA
тоже живут, одна — нет: вместе с ней пропадают выпуск и продление сертификатов.

Топология для трёх-четырёх реплик — полная сетка и для суффикса домена, и для
`ca`. Проверка: `ipa topologysegment-find --all` (отдельно для `domain` и `ca`).

Перед LDAP и Kerberos не ставят обычный TCP-балансировщик. Клиенты находят
реплики через DNS SRV и явный список в SSSD. Балансировщик прячет, какая реплика
жива, и ломает репликацию, если через него же ходят соглашения.

CA renewal master один. Перед обслуживанием этой реплики роль переносят:

```bash
ipa config-show
ipa config-mod --ca-renewal-master-server=ipa2.example.test
```

Память реплики с CA и DNS — от 8 ГиБ, без жёсткого overcommit. Каталог на полном
диске перестаёт фиксировать транзакции; отдельные тома `/var` и
`/var/lib/dirsrv` плюс алерт на 70 % полезнее любого тюнинга. Часы общие:
расхождение больше пяти минут ломает Kerberos. Swap лучше оставить небольшим,
чтобы OOM killer не приходил раньше, чем система успеет сбросить кэш.

Обслуживание только по одной реплике. Пока `replication_bad` не вернётся в ноль,
следующую не трогают. `re-initialize` и восстановление бэкапа поверх живой
реплики — уже разбор аварии, не плановая операция.

Клиентам нужны минимум два DNS с SRV-записями IPA. В SSSD список
`_srv_, ipa1, ipa2, ipa3`: сначала DNS, при отказе DNS — явные хосты.
`cache_credentials = True` оставляет уже открытые сессии, когда IPA недоступна.
Это делает `clients.yml`.

## Что меняет роль на реплике

Сразу, без рестарта каталога:

- `nsslapd-idletimeout=3600` — рвёт зависшие соединения, из-за которых кончается таблица коннектов;
- `nsslapd-ioblocktimeout=20000` — короткая пауза диска не обрывает репликацию;
- `nsslapd-changelogmaxage=30d` — выключенная реплика ещё может догнать поток, а не требовать полную переинициализацию;
- потолок логов dirsrv, только если сейчас они безлимитные;
- `oom_score_adj` у живых процессов: ядро раньше убьёт что-то другое, чем `ns-slapd` или KDC.

Записывается сейчас, действует после следующего старта процесса:

- лимит файловых дескрипторов 65535 и backlog 256;
- `Restart=on-failure` не больше трёх раз за десять минут, чтобы порченая база не ушла в бесконечный цикл;
- coredump `ns-slapd` до 2 ГиБ — есть что открыть в `coredumpctl` после падения.

Наблюдаемость:

- журнал systemd переживает перезагрузку и ограничен 2 ГиБ;
- каждые 15 минут `/usr/local/sbin/ipa-resilience-state` пишет `/var/log/ipa/state/latest.txt`;
- раз в сутки и через 10 минут после загрузки — `ipa-healthcheck` в `/var/log/ipa/healthcheck/`;
- если снимок не `OK`, юнит виден в `systemctl --failed`.

Локальный `ipa-backup` выключен. Он легко забивает `/var`, а это как раз тот
сбой, от которого роль страхует. Включайте его только вместе с выносом копий
на другую машину: `freeipa_resilience_backup_enabled: true`.

## Запуск

ОС реплик: Rocky Linux 9 или AlmaLinux 9, одна и та же мажорная версия, от 8 ГиБ
RAM. Имена в inventory — постоянные FQDN.

```bash
cp inventory/hosts.yml.example inventory/hosts.yml
cp inventory/secrets.yml.example inventory/secrets.yml
```

В `inventory/hosts.yml` замените имена и `freeipa_ip_address`. В
`inventory/group_vars/ipa_servers.yml` замените `freeipa_domain` и задайте
`freeipa_dns_forwarders`. В `inventory/secrets.yml` задайте два разных пароля,
затем:

```bash
ansible-vault encrypt inventory/secrets.yml
ansible-playbook install.yml --ask-vault-pass
```

`install.yml` ставит первую реплику с CA и DNS, затем остальные по одной, на
каждой включает укрепление. Пока реплика одна, снимок состояния покажет
`single_server` — это ожидаемо до конца playbook.

Повторный запуск `install.yml` пропускает уже установленные хосты. Если
установка оборвалась, на этой машине сначала `ipa-server-install --uninstall -U`.

Дальше, когда IPA уже стоит:

```bash
ansible-playbook site.yml
ansible-playbook clients.yml
```

`clients.yml` — после правки `inventory/group_vars/ipa_clients.yml`. Первый
прогон `site.yml` идёт по одной реплике и в конце ждёт healthcheck. Чтобы только
разложить таймеры:

```bash
ansible-playbook site.yml -e freeipa_resilience_run_healthcheck_now=false
```

Лимит дескрипторов подхватится при штатном рестарте. Осознанный поочерёдный
рестарт, отдельно от установки:

```bash
ansible-playbook site.yml -e freeipa_resilience_rolling_restart=true
```

Параметры лежат в `roles/freeipa_resilience/defaults/main.yml`.
`freeipa_install_debug=true` печатает командную строку установщика вместе с
паролями, для обычного прогона оно не нужно.

## Как разбирать состояние

На реплике, начиная со сводки:

```bash
cat /var/log/ipa/state/latest.txt
cat /var/log/ipa/healthcheck/latest.summary.txt
systemctl --failed
systemctl list-timers 'ipa-resilience-*'
```

Та же памятка после прогона лежит в
`/usr/local/share/doc/freeipa-resilience/RUNBOOK.txt`.

`attention_reasons` в сводке:

| Причина | Куда смотреть |
| --- | --- |
| `service:` | `journalctl -u <unit> -n 200` и `ipactl status` |
| `chrony` | `chronyc tracking`. Пока время не сошлось, Kerberos будет врать |
| `low_memory`, `oom` | `journalctl -k -g oom`, нехватка RAM или утечка |
| `dirsrv_fds` | таблица соединений; после рестарта поможет новый лимит дескрипторов |
| `disk:` | место на томе. Пока раздел полный, каталог лучше не рестартовать пачкой |
| `replication` | в отчёте строки `BAD`: сеть до соседней реплики или она сама лежит |
| `single_server` | реплика одна, отказ этой машины роняет домен |
| `certs` | `getcert list`, часто рядом мёртвый `pki-tomcatd` или renewal master |
| `dirsrv_down` | нет сокета LDAPI, каталог не запущен |

Снимки за 14 дней остаются в `/var/log/ipa/state/`. Для передачи наружу:
`sos report -o ipa,dirsrv,krb5,httpd,logs`.

Пока не ясно, какая реплика новее, не делают `ipa-replica-manage re-initialize`
и не заливают бэкап поверх живого каталога.
