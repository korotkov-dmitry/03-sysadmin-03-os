# Домашнее задание к занятию "3.3. Операционные системы, лекция 1"

## 1. Какой системный вызов делает команда `cd`? В прошлом ДЗ мы выяснили, что `cd` не является самостоятельной  программой, это `shell builtin`, поэтому запустить `strace` непосредственно на `cd` не получится. Тем не менее, вы можете запустить `strace` на `/bin/bash -c 'cd /tmp'`. В этом случае вы увидите полный список системных вызовов, которые делает сам `bash` при старте. Вам нужно найти тот единственный, который относится именно к `cd`.
`strace /bin/bash -c 'cd /tmp'
...
chdir("/tmp")
...`
## 2. Попробуйте использовать команду `file` на объекты разных типов на файловой системе. Например:
    ```bash
    vagrant@netology1:~$ file /dev/tty
    /dev/tty: character special (5/0)
    vagrant@netology1:~$ file /dev/sda
    /dev/sda: block special (8/0)
    vagrant@netology1:~$ file /bin/bash
    /bin/bash: ELF 64-bit LSB shared object, x86-64
    ```
    Используя `strace` выясните, где находится база данных `file` на основании которой она делает свои догадки.
    
База данных - `openat(AT_FDCWD, "/usr/share/misc/magic.mgc", O_RDONLY) = 3`.

Дополнительно осуществляет поиск:

    `stat("/home/vagrant/.magic.mgc", 0x7fff4886c6b0) = -1 ENOENT (No such file or directory)
    stat("/home/vagrant/.magic", 0x7fff4886c6b0) = -1 ENOENT (No such file or directory)
    openat(AT_FDCWD, "/etc/magic.mgc", O_RDONLY) = -1 ENOENT (No such file or directory)
    stat("/etc/magic", {st_mode=S_IFREG|0644, st_size=111, ...}) = 0
    openat(AT_FDCWD, "/etc/magic", O_RDONLY) = 3`
## 3. Предположим, приложение пишет лог в текстовый файл. Этот файл оказался удален (deleted в lsof), однако возможности сигналом сказать приложению переоткрыть файлы или просто перезапустить приложение – нет. Так как приложение продолжает писать в удаленный файл, место на диске постепенно заканчивается. Основываясь на знаниях о перенаправлении потоков предложите способ обнуления открытого удаленного файла (чтобы освободить место на файловой системе).
    Получаем список всех процессов
    
    `ps aux
    ...
    vagrant     2824  0.0  0.9  24348  9788 pts/1    S+   03:29   0:00 vi test_deleted 
    ...`
    
    Получаем список используемых файлов процессом
    `lsof -p 2824
    ...
    vi      2824 vagrant    4u   REG  253,0    12288 231088 /home/vagrant/.test_deleted.swp здесь должен быть признак (deleted)
    ...`
    
    Перенаправление
    `echo '123' >/proc/2824/fd/4`
## 4. Занимают ли зомби-процессы какие-то ресурсы в ОС (CPU, RAM, IO)?
Зомби-процессы не занимают памяти (как процессы-сироты), но блокируют записи в таблице процессов.
## 5. В iovisor BCC есть утилита `opensnoop`:
    ```bash
    root@vagrant:~# dpkg -L bpfcc-tools | grep sbin/opensnoop
    /usr/sbin/opensnoop-bpfcc
    ```
    На какие файлы вы увидели вызовы группы `open` за первую секунду работы утилиты? Воспользуйтесь пакетом `bpfcc-tools` для Ubuntu 20.04. Дополнительные [сведения по установке](https://github.com/iovisor/bcc/blob/master/INSTALL.md).
    
    `vagrant@vagrant:~$ dpkg -L bpfcc-tools | grep sbin/opensnoop
    /usr/sbin/opensnoop-bpfcc
    vagrant@vagrant:~$ sudo /usr/sbin/opensnoop-bpfcc
    PID    COMM               FD ERR PATH
    784    vminfo              4   0 /var/run/utmp
    567    dbus-daemon        -1   2 /usr/local/share/dbus-1/system-services
    567    dbus-daemon        18   0 /usr/share/dbus-1/system-services
    567    dbus-daemon        -1   2 /lib/dbus-1/system-services
    567    dbus-daemon        18   0 /var/lib/snapd/dbus-1/system-services/
    ^Cvagrant@vagrant:~$`

## 6. Какой системный вызов использует `uname -a`? Приведите цитату из man по этому системному вызову, где описывается альтернативное местоположение в `/proc`, где можно узнать версию ядра и релиз ОС.
Системный вызов `uname()`

Цитата `Part of the utsname information is also accessible via /proc/sys/kernel/{ostype, hostname, osrelease, version, domainname}.`

## 7. Чем отличается последовательность команд через `;` и через `&&` в bash? Например:
    `bash
    root@netology1:~# test -d /tmp/some_dir; echo Hi
    Hi
    root@netology1:~# test -d /tmp/some_dir && echo Hi
    root@netology1:~#`
    
    Есть ли смысл использовать в bash `&&`, если применить `set -e`?
    
    `&&` логический оператор. 
    `;` разделитель команд.
    
    `test -d /tmp/some_dir && echo Hi` `echo` выполнится, если команда `test` выполнится успешно.
   `set -e` установка или снятие значений параметров оболочки. Использование с `&&` не имеет смысла, т.к. с `-e` произойдет немедленный выход, если команда завершается с ненулевым статусом. 
## 8. Из каких опций состоит режим bash `set -euxo pipefail` и почему его хорошо было бы использовать в сценариях?

    `-e` Немедленный выход, если команда завершается с ненулевым статусом. 
    `-u` При подстановке обрабатывать неустановленные переменные как ошибку.
    `-x` Печатать команды и их аргументы по мере их выполнения. 
    `-o option-pipefail` возвращаемое значение конвейера - это статус последней команды для выхода с ненулевым статусом или ноль, если ни одна команда не завершилась с ненулевым статусом.

    Для сценария увеличивает детальность логирования. 
    Прервет сценарий при возникновении ошибки, кроме завершающей команды.
## 9. Используя `-o stat` для `ps`, определите, какой наиболее часто встречающийся статус у процессов в системе. В `man ps` ознакомьтесь (`/PROCESS STATE CODES`) что значат дополнительные к основной заглавной буквы статуса процессов. Его можно не учитывать при расчете (считать S, Ss или Ssl равнозначными).

    `Ss` - неактивные процессы;
    `R+` - выполняющиеся в группе приоритетных.
    
    Дополнительные к заглавной букве - это дополнительные значения состояния процесса:
    `<` - высокий приоритет;
    `N` - низкий приорит;
    `L` - имеет страницы, заблокированные в памяти;
    `s` - является лидером сеанса;
    `l` - является многопоточным;
    `+` - находится в группе приоритетных процессов.    
