# Natch Quickstart

*(актуально на момент 21.05.2022)*

## Введение

Данное руководство является кратким курсом по освоению основных принципов использования системы определения ширины и глубины поверхности атаки Natch (разработчик - [ИСП РАН](https://www.ispras.ru/)). В руководстве рассмотрены вопросы создания виртуализованных сред выполнения объекта оценки (далее - ОО) в формате [QEMU](https://wiki.qemu.org/Main_Page), запуска данных сред под контролем Natch, анализа информации о движении помеченных данных, получаемой Natch от контролируемой виртуализованной среды.

## 1. Общие вопросы

### 1.1. Общие принципы работы Natch

Natch (Network Application Tainting Can Help) - это инструмент для определения поверхности атаки, основанный на полносистемном эмуляторе Qemu. Основная функция Natch - получение списка модулей (исполняемых файлов и динамических библиотек) и функций, используемных системой во время выполнения задачи. Natch представляет собой набор плагинов для эмулятора Qemu.

Общие принципы работы Natch, доступные плагины, команды управления Natch и их параметры представлены в веб-странице руководства `Natch v.1.3 — QEMU documentation.html`, доступном в комплекте поставки.

#### Комплект поставки

Комплект поставки Natch доступен в облаке, ниже приведены ссылки на доступные релизы (рекомендуется использовать наиболее актуальный релиз):

**Актуальный релиз:**

[Natch v.1.3](https://nextcloud.ispras.ru/index.php/s/Natch_v.1.3/download?path=%2F&downloadStartSecret=yyiov6rvlj)

Предыдущие релизы:

[Natch v.1.2.1](https://nextcloud.ispras.ru/index.php/s/8YSBjPSCMzAttWx)

[Natch v.1.2](https://nextcloud.ispras.ru/s/Sd439xDgoyzLPTt)


### 1.2. Общие принципы подготовки виртуализированной среды в формате QEMU

Процесс подготовки виртуализированной среды выполнения ОО в общем случае состоит из следующих последовательных шагов:
 - создание образа эмулируемой операционной системы в формате диска [qcow2](https://en.wikipedia.org/wiki/Qcow) на основе базового дистрибутива ОС. Формат qcow2 позволяет эффективно формировать снэпшоты состояния файловой системы в произвольный момент выполнения виртулизованной среды функционирования;
 - сборка дистрибутива ОО с требуемыми параметрами, в частности с генерацией и сохранением отладочных символов;
 - помещение собранного дистрибутива ОО в виртуализированную среду выполнения; - 
 - подготовка команд запуска QEMU, обеспечивающих эмуляцию аппаратной составляющей среды функционирования, загрузку и выполнение компонент Natch. 

Процесс подготовки виртуализированной среды выполнения ОО в значительной степени сходен с процессом подготовки виртуализированной среды для анализа с помощью инструмента динамического анализа помеченных данных [Блесна](https://www.ispras.ru/technologies/blesna/) (разработчик - [ИСП РАН](https://www.ispras.ru/)), с точностью до подготовки команд запуска QEMU.

Процесс подготовки виртуализированной среды выполнения ОО рекомендуется выполнять в хостовой системе, допускающей запуск QEMU в режиме пользовательской виртуализации (ключ `-enable-kvm`) - это существенно ускорит процесс, скорость работы в режиме аппаратной виртуализации более чем на порядок превосходит работу в режиме полносистемной эмуляции. Проверить доступность данного режима в вашей хостовой системе (равно как и установить kvm-модули в вашу систему) можно опираясь на следующую [статью](https://phoenixnap.com/kb/ubuntu-install-kvm) с помощью команды:

```bash
sudo kvm-ok
```

при написании данной статьи в моей хостовой системе такая возможность отсутствовала, но менять конфигурации хостовой ВМ VirtualBox мне не хотелось

```bash
[sudo] password for user: 
INFO: Your CPU does not support KVM extensions
KVM acceleration can NOT be used
```

### 1.3. Пример подготовки виртуализированной среды в формате QEMU

#### Подготовка хостовой системы

Подготовим Linux-based рабочую станцию (далее - хост), поддерживающую графический режим выполнения (QEMU демонстрирует вывод эмулируемой среды выполнения в отдельном графическом окне, следовательно нам необходим графический режим). Рабочая станция может быть реализована в формате виртуальной машины. Данное руководство описывает действия пользователя, работающего в виртуальной машине VirtualBox (4 ядра, 8 ГБ ОЗУ) с установленной ОС [Ubuntu20:04](https://releases.ubuntu.com/20.04/ubuntu-20.04.3-desktop-amd64.iso) (desktop-конфигурация, обновить пакеты при установке). 

Установим требуемое системное ПО, в т.ч. QEMU (*Подсказка: данная инсталляция требуется не для запуска Natch, но для создания образов ВМ на произвольном хосте. Natch содержит в своём составе требуемую для работы версию QEMU*): 
```bash
sudo apt install  -y curl qemu-system gcc g++ 
```

Скачаем на хост выбранный базовый дистрибутив ОС. Поскольку виртуальные машины QEMU в режиме полносистемной эмуляции (анализ выполняется именно в таком режиме (параметр `--enable-kvm` в строке запуска qemu должен отсутствовать), поскольку нам требуется полная эмуляция процессора и, частично, периферии для сбора максимально полной отладочной информации) значительно замедляют выполнение анализируемой среды функционирования, рекомендуется использовать минимальный образ - уменьшение числа установленных, стартующих при запуске служб, позволит значительно сократить нагрузку на процессор и сократить время, требуемое на запись и анализ трасс в режиме полносистемной эмуляции. В данном примере в качестве базового выбран образ [lubuntu](https://lubuntu.net/downloads/) - скачаем его на хост, предварительно создав на хосте какой-нибудь каталог для дальнейших экспериментов:

```bash
cd ~ && mkdir natch_quickstart && cd natch_quickstart
curl -o lubuntu-18.04-alternate-amd64.iso  'http://cdimage.ubuntu.com/lubuntu/releases/18.04/release/lubuntu-18.04-alternate-amd64.iso'
```

и выполним создание виртуальной машины (я опирался на вот это [руководство](https://graspingtech.com/ubuntu-desktop-18.04-virtual-machine-macos-qemu/), но их много разных доступно в интернете).

Проверим установку ВМ:
```bash
qemu-system-x86_64 --version
QEMU emulator version 4.2.1 (Debian 1:4.2-3ubuntu6.19)
Copyright (c) 2003-2019 Fabrice Bellard and the QEMU Project developers
```

Создадим образ диска в формате `qcow2`, с именем `lubuntu.qcow2` и размером `20 ГБайт`.
```bash
qemu-img create -f qcow2 lubuntu.qcow2 20G
Formatting 'lubuntu.qcow2', fmt=qcow2 size=21474836480 cluster_size=65536 lazy_refcounts=off refcount_bits=16
ll
total 8100768
drwxrwxr-x  4 user user       4096 янв 30 20:40 ./
drwxr-xr-x 25 user user       4096 янв 30 20:40 ../
-rw-rw-r--  1 user user  751828992 янв 30 20:34 lubuntu-18.04-alternate-amd64.iso
-rw-r--r--  1 user user     196928 янв 30 20:08 lubuntu.qcow2
```

Создадим скрипт запуска нашей ВМ `run.sh` - скрипт (равно как и дальнейшие скрипты в данном руководстве) достаточно объемный, поэтому желаетельно именно сохранять его в виде отдельного файла, допускающего удобное внесение изменений. Для тех, кто сталкивается с синтаксисом QEMU впервые, настоятельно рекомендуется ознакомиться с основными командами, кратко расписанными в веб-странице руководства `Natch v.1.1 — QEMU documentation.html`, либо подробно расписанными в официальной [документации QEMU](https://www.qemu.org/docs/master/system/invocation.html).

```bash
qemu-system-x86_64 \
-hda lubuntu.qcow2  \
-m 4G \
-monitor stdio \
-netdev user,id=net0 \
-device e1000,netdev=net0 \
-cdrom lubuntu-18.04-alternate-amd64.iso
```

и выполним его:

```bash
./run.sh
```

после чего увидим знакомое нам графическое окно установки легковесной версии ubuntu. Выполним установку версии с минимально желаемым набором параметров (при установке будет создан пользователь с уч. данными test:test). *Подсказка: чтобы вывести курсор мыши из открытого графического окна ВМ нажмите Ctrl+Alt+G*

После завершения установки удалим из скрипта запуска `run.sh` указание подключения cdrom - для дальнешей работы он нам не потребуется

```bash
#-cdrom ubuntu-18.04.3-desktop-amd64.iso
```
Наш образ среды функционирования готов к работе - в частности к установке в него пресобранного **с символами** прототипа объекта оценки.

#### Сборка прототипа объекта оценки и генерация map-файлов средствами компилятора

В качестве прототипа объекта оценки будем использовать популярную программу wget. Скачаем её исходные тексты с репозитория:
```bash
curl -o wget-1.21.2.tar.gz  'https://ftp.gnu.org/gnu/wget/wget-1.21.2.tar.gz'
tar -xzf wget-1.21.2.tar.gz && cd wget-1.21.2
```

Для выполнения [*классического*](https://thoughtbot.com/blog/the-magic-behind-configure-make-make-install) подготовительного скрипта `configure`, генерирующего make-файл, потребуется установить ряд дополнительных зависимостей (скрипт выведет их наименования в случае неудачного завершения), в том числе для моей системы потребовалось (*Подсказка: поскольку я собираюсь собирать wget из исходников мне нужен комплект заголовочных файлов, доступный как раз в dev-версии пакета gnutls*):

```bash
sudo apt install  -y gnutls-dev gnutls-bin
```

Успешное выполнение скрипта `configure` с определенными предустановками позволит в т.ч. установить параметры компилятора - требуется для генерации [map-файлов](https://stackoverflow.com/questions/22199844/what-are-gcc-linker-map-files-used-for), сохраняющих информацию об отладочных символах. Выполним скрипт: 

```bash
CFLAGS='-g0 -Xlinker -Map=output.map' ./configure && \
make
```

и проверим, что map-файлы создались. Данные файлы - в составе общего каталога сборки wget - потребуются нам на хосте, при запуске Natch.

```bash
find . -name *.map
./src/output.map
./output.map
```

#### Генерация map-файлов сторонними инструментов

Получение map-файлов для исполняемого файла, собранного с отладочными символами, возможно с помощью сторонних инструментов. Это может быть актуально в тех случаях, когда сборочный конвейер недоступен, либо получение от сборочного конвейера map-файлов в поддерживаемом Natch формате невозможно (например, использование специфического компилятора/компоновщика). Сгенерируем map-файлы с использованием бесплатной версии дизассемблера [IDA](https://hex-rays.com/ida-free/) - необходимо скачать установочный комплект по указанной ссылке, возможно в систему придётся доустановить библотеки Qt `apt install -y qt5-default`.

После установки IDA необходимо запустить её, открыть интересующий нас исполняемый файл (в нашем случае это `wget`)

![image](https://user-images.githubusercontent.com/46653985/152351389-7cd54129-c087-4dc3-8bf6-be500f8193c3.png)

пройти процедуру генерации map-файла

![image](https://user-images.githubusercontent.com/46653985/152351322-d92d16e0-5650-4c96-a61a-3abeb992b18e.png)


выбрав экспорт всей возможной информации

![image](https://user-images.githubusercontent.com/46653985/152351245-d3a3510f-adba-4919-adfb-be081a6800e8.png)


после чего убедиться, что map-файл появился в файловой системе

```bash
ll src | grep .map
-rw-rw-r--  1 user user  564686 фев  3 16:11 wget.map
```

#### Перенос прототипа объекта оценки в образ ВМ

Существуют различные способы помещения обмена информацией между хостом и виртуальной машиной. Воспользуемся подходом, основанным на использовании [nbd-сервера QEMU](https://manpages.debian.org/testing/qemu-utils/qemu-nbd.8.en.html), позволяющим [смонтировать](https://gist.github.com/shamil/62935d9b456a6f9877b5) созданный ранее qcow2-диск ВМ в файловую систему хостовой ОС. Для выполнения монтирования диск не должен быть задействован (виртуальная машина должна быть выключена).

Загрузим NBD-драйвер в ядро хостовой ОС:

```bash
modprobe nbd max_part=8
```

Смонтируем наш образ диска как сетевое блочное устройство:
```bash
sudo qemu-nbd --connect=/dev/nbd0 lubuntu.qcow2
```

Определим число разделов на устройстве:
```bash
fdisk /dev/nbd0 -l
Disk /dev/nbd0: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe6ea1316

Device      Boot Start      End  Sectors Size Id Type
/dev/nbd0p1 *     2048 41940991 41938944  20G 83 Linux

```

Смонтируем раздел в какой-либо каталог хостовой ОС (например, традиционно, в mnt)
```bash
sudo mount /dev/nbd0p1 /mnt/
ls /mnt
bin   dev  home        initrd.img.old  lib64       media  opt   root  sbin  swapfile  tmp  var      vmlinuz.old
boot  etc  initrd.img  lib             lost+found  mnt    proc  run   srv   sys       usr  vmlinuz
```

Поместим прототип ОО на смонтированный раздел:
```bash
sudo cp -r wget-1.21.2 /mnt/ && ls /mnt
bin   dev  home        initrd.img.old  lib64       media  opt   root  sbin  swapfile  tmp  var      vmlinuz.old
boot  etc  initrd.img  lib             lost+found  mnt    proc  run   srv   sys       usr  vmlinuz  wget-1.21.2
```

Отмонтируем диск:
```bash
sudo umount /mnt/
sudo qemu-nbd --disconnect /dev/nbd0
sudo rmmod nbd
```

#### Тестирование виртуализированной среды функционирования ОО

Запускаем ВМ скриптом `run.sh` с учетом отключенного ранее cdrom, дожидаемся загрузки ОС ВМ, авторизуемся в ОС от имени учетной записи test, пробуем выполнить обращение к произвольному сетевому ресурсу с помощью собранной нами версии wget:

```bash
cd /wget-1.21.2 && sudo ./wget ispras.ru
```

В результате вы должны увидеть приблизительно следующую картину в графическом окне QEMU, свидетельствующую о том, что ОО корректно выполняется в среде функционирования и сетевая доступность для ВМ обеспечена:

![image](https://user-images.githubusercontent.com/46653985/151779302-c2d59bd1-aed1-45ce-8f00-2702380a3157.png)


## 2. Обучающие примеры 

### 2.1 Анализ образа системы, содержащего пресобранную с символами программу wget2

Для выполнения данного примера потребуется:
- рабочая станция под управлением ОС Linux (традиционно Ubuntu 20.04). Установка пакета **qemu-system** не требуется, нужная версия входит в дистрибутив Natch;
- актуальный [дистрибутив](#комплект-поставки) Natch;
- подготовленный разработчиком [тестовый набор]([https://nextcloud.ispras.ru/index.php/s/eCsXcM6QQgxRLWQ](https://nextcloud.ispras.ru/index.php/s/testing_materials)), включающий в себя пресобранную с символами и map-файлами версию wget2 и тестовый образ на базе Debian10, содержащий пресобранную версию wget2. 

#### Получение образа и дистрибутива
```bash ву  
# Получение дистрибутива Natchmv
curl -o Natch.zip 'https://nextcloud.ispras.ru/index.php/s/Natch_v.1.3/download?path=%2F&downloadStartSecret=yyiov6rvlj' && \
curl -o Natch_testing_materials.zip 'https://nextcloud.ispras.ru/index.php/s/testing_materials/download?path=%2F&downloadStartSecret=mv78e6aiu1'

unzip Natch.zip && \
unzip Natch/Docs/docs.zip -d . && \
unzip Natch/Bins/qemu_plugins_2004_natch_release_latest.zip -d . && \
unzip Natch/Bins/libs_2004_natch_release_latest.zip -d . && \
rm -f Natch.zip && \
rm -rf Natch &&rm -rf 

unzip Natch_testing_materials.zip && \
mv Natch_testing_materials/* . && \
unzip -u wget.zip && \
rm -rf Natch_testing_materials && \
rm Natch_testing_materials.zip && \
rm wget.zip
```

Проверим, что всё на месте:
```bash
ls -la
total 7150948
drwxrwxr-x  6 tester tester       4096 мая 21 12:14 .
drwxr-xr-x 19 tester tester       4096 мая 10 10:30 ..
-rw-r--r--  1 tester tester     197632 апр 19 23:08 debian10_wget2.diff
-rw-r--r--  1 tester tester 7322337280 апр 21 09:52 debian10_wget2.qcow2
drwxrwxr-x  3 tester tester       4096 апр 22 10:52 docs
drwxr-xr-x  2 tester tester       4096 апр 21 16:25 libs
drwxr-xr-x  9 tester tester       4096 апр 21 16:25 qemu_plugins_2004_natch_release
drwxrwxr-x  3 tester tester       4096 дек 23 15:17 wget2
-rw-rw-r--  1 tester tester        208 апр 19 23:00 wget_api.cfg
```

В каталоге `docs` размещается веб-страница руководства `Natch v.1.3 — QEMU documentation.html`. В каталоге `libs` размещаются используемые Natch библиотеки (подключаются с использованием стандартного механизма [preload](https://www.baeldung.com/linux/ld_preload-trick-what-is#:~:text=The%20LD_PRELOAD%20trick%20is%20a,a%20collection%20of%20compiled%20functions.) при запуске qemu-system). В каталоге `qemu_plugins...` помещаются собственно исполняемые файлы Natch. Подготовленный образ в формате qcow2, каталог с пресобранной версией wget2 а также файл конфигурации `wget_api.cfg` (будет рассмотрен ниже) помещены в корень рабочего каталога (размещение не принципиально).

#### Установка python-библиотек, обеспечивающих работоспособность скриптов
```bash
cd qemu_plugins_2004_natch_release/bin/natch_scripts/
pip3 install -r requirements.txt
```

#### Настройка Natch

Процесс настройки Natch состоит из четырёх этапов (предназначение файлов конфигураци и их параметров **подробно расписано в документации**):

##### Этап 1. Заполнение конфигурации маппинга исполняемых модулей

Заполнение данного файла позволяет повысить вероятность распознавания Natch имен модулей и функций, исполняемых в виртуализированной среде. В комплекте поставки тестового примера имеется предзаполненный файл `wget_api.cfg`, который содержит маппинги для пресобранных с символами исполняемого файла wget2 и разделяемой библиотеки libwget.so.1.0.0, расположенных в каталоге `natch_quickstart\wget2` хостовой системы в формате непосредственно указания пути к файлу и соответствующего ему map-файла (альтернативный подход - указание каталога со всеми образами исполняемых файлов - описан в документации). Первоначально файл имеет следующий вид:

```bash
cat wget_api.cfg 
[Image1]
path=wget2/wget2
map=wget2/wget2.map
textstart=0xC000

[Image2]
path=wget2/lib/libwget.so.1.0.0
map=wget2/lib/libwget.so.1.0.0.map
textstart=0xe000
```

Модифицируем файл, указав абсолютные пути (для надёжности) и удалив описание секции `textstart` (избыточно, поскольку адреса функции нашего исполняемого файла заданы в относительном формате, что допускает их автоматическое вычисление без дополнительной информации - см. документацию п. 2.2): 

```bash
cat ../wget_api.cfg 
[Image1]
path=~/natch_quickstart/wget2/wget2
map=~/natch_quickstart/wget2/wget2.map

[Image2]
path==~/natch_quickstart/wget2/lib/libwget.so.1.0.0
map==~/natch_quickstart/wget2/lib/libwget.so.1.0.0.map
```

Заполнение конфигурационного файла можно автоматизировать, указав на вход скрипту `qemu_plugins_2004_natch_release/bin/natch_scripts\module_config.py`, описанному в п. 2.2 документации, имя каталога на хосте, содержащего исполняемые файлы и map-файлы (имена map-файлов должны совпадать с именами соответствующих исполняемых файлов).

##### Этап 2. Формирование скриптового и конфигурациионного окружения

С целью решения задачи автоматизации порождения типовых скриптов управления и основного файла конфигурации Natch создан скрипт `natch_run.py`. На вход данному скрипту необходимо подать некоторые параметры, такие как:
- имя образа qcow2 (обязательный параметр)
- ряд опциональных параметров (выделяемый ВМ объем RAM, специальная версия ядра Линукс), в т.ч. включающих/выключающих некоторые аналитические возможности Natch (подробнее см. документацию п. 3.1) 

Параметр пути к образу qcow2 допускается задавать в виде переменной скрипта, воспользуемся этим подходом:

```bash
cd qemu_plugins_2004_natch_release/bin/natch_scripts && \
./natch_run.py  ~/natch_quickstart/debian10_wget2.qcow2
```

И несколько нажатий Enter - параметры по умолчанию нас устраивают, за исключение принудительно включаемых генераторов графов:

```bash
Image: /home/tester/natch_quickstart/debian10_wget2.qcow2
Common options
Enter RAM size (in Gb, e.g. 4): 
The default value is set to 4Gb
Do you want include launch Natch into command line? [Y/n] 
Natch options
Enter name of config file or press Enter for using default name: 
Do you want enable task_graph? [y/N]  y        <<-- Включить отрисовку графа задач
Do you want enable module_graph? [y/N]  y        <<-- Включить отрисовку графа модулей
Only for record mode
Do you want enable net_log? [Y/n]  
Now we will try to create overlay
Overlay created        <<-- Успешно создан слой для виртуального диска
Enter custom options (with '-'): 

Settings completed!

Here is your command line one:
```

Мы должны увидеть лог, содержащий сообщения о том, что скрипты запуска Natch подготовлены успешно:

```bash
...
File '/home/tester/natch_quickstart/qemu_plugins_2004_natch_release/bin/natch_scripts/natch_config.cfg' created. You must edit it before using Natch.

After checking config file you can launch:
	just Qemu with help './run.sh'
	Qemu in record mode with help'./run_record.sh'
	Qemu in replay mode with help'./run_replay.sh'

```
а также три скрипта: 
- скрипт первоего - настроечного - запуска Natch (подробнее см. документацию п. 3.3)
- скрипт запуск Natch в режиме записи трассы для последующего анализа (подробнее см. документацию п. 3.4)
- скрипт запуска Natch в режиме воспроизведения и анализа трассы (подробнее см. документацию п. 3.5) 

Отредактируем сгенерированный основной конфигурационный файл  Natch `natch_config.cfg` в соответствии с рекомендациями (подробнее см. документацию п. 2.1). В качестве порта - источника помеченных данных - оставим порты 80 и 443 (в ходе теста будем выполнять обращение к сайту по HTTP-протоколу (80), после чего будет выполнено автоматическое перенаправление на HTTPS-версию страницы (443)). Раскомментируем секцию Taint для сохранения сведений о взаимодействии анализируемого процесса с помеченными данными. Укажем путь к файлам, содержащим map-файлы в секции Modules.

```bash
# Natch config sample
 
 [Ports]
 in=80,443
 out=80,443

 [Taint]
 threshold=250
 on=true

 [Modules]
  config=~/natch_quickstart/wget_api.cfg
# log=api_taint_log.txt

 [Tasks]
  config=task_config.ini
# file=processes.log

# [Syscalls]
# config=x86_64_Linux.cfg

# [Netlog]
# log=netpackets.log
# tlog=tnetpackets.log

# [Icount]
# steps=1111111111;5555555555



```

Выполним настроечный запуск Natch с помощью сформированного скрипта `run.sh`, не забыв указать путь к ряду библиотек с помощью уже упоминавшегося механизма LD_PRELOAD:

```bash
LD_LIBRARY_PATH=~/natch_quickstart/libs/ ./run.sh
```
Появятся картинка и текст наподобие:

![image](https://user-images.githubusercontent.com/46653985/164391795-e43f00b4-80a8-415d-bda0-a1531671e17a.png)

Через 2-3 минуты процесс завершится успешно (_нюанс - в моем случае при запуске qemu внутри виртуальной машины VBox периодически отмечается подвисание GUI хостовой ОС. Лечится в частности минимизацией/максимизацией окна виртуальной машины VBox_):

![image](https://user-images.githubusercontent.com/46653985/164392153-7608f8d8-e5dc-44a6-accf-767bd1b43e2e.png)

Теперь всё готово к записи трассы.

##### Этап 3. Запись трассы


Теперь всё готово к анализу трассы.

##### Этап 4. Анализ трассы












