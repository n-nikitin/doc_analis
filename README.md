# Natch Quickstart

*(актуально на момент бета-релиза Natch 2.0)*

## Оглавление

  * [0. Введение](#0---------)
  * [1. Общие вопросы](#1--------------)
    + [1.1. Общие принципы работы Natch](#11-----------------------natch)
      - [1.1.1. Комплект поставки](#111------------------)
    + [1.2. Общие принципы подготовки виртуализованной среды в формате QEMU](#12------------------------------------------------------------qemu)
    + [1.3. Пример подготовки виртуализованной среды в формате QEMU](#13----------------------------------------------------qemu)
      - [1.3.1. Подготовка хостовой системы](#131----------------------------)
      - [1.3.2. Сборка прототипа объекта оценки](#132--------------------------------)
      - [1.3.3. *Генерация map-файлов средствами компилятора*](#133------------map-------------------------------)
      - [1.3.4. *Генерация map-файлов сторонними инструментами*](#134------------map---------------------------------)
      - [1.3.5. Перенос прототипа объекта оценки в образ ВМ](#135--------------------------------------------)
      - [1.3.6. Тестирование виртуализированной среды функционирования ОО](#136----------------------------------------------------------)
  * [2. Обучающие примеры](#2------------------)
    + [2.1. Анализ образа системы, содержащего тестовые комплекты пресобранных исполняемых файлов](#21--------------------------------------------------------------------------------------)
      - [2.1.1. Получение образа и дистрибутива](#211--------------------------------)
      - [2.1.2. Установка Natch и Snatch](#212-----------natch---snatch)
      - [2.1.3. Настройка Natch для работы с тестовым образом ОС](#213-----------natch---------------------------------)
        * [2.1.3.1. Автоматизированная настройка](#2131-----------------------------)
        * [2.1.3.2. Дополнительная ручная настройка](#2132--------------------------------)
        * [2.1.4. Запись трассы](#214--------------)
        * [2.1.5. Воспроизведение трассы](#215-----------------------)
        * [2.1.6. Анализ трассы в ручном режиме](#216------------------------------)

## 0. Введение

Данное руководство является кратким курсом по освоению основных принципов использования системы определения ширины и глубины поверхности атаки [Natch](https://www.ispras.ru/technologies/natch/) (разработчик - [ИСП РАН](https://www.ispras.ru/)). В руководстве рассмотрены вопросы создания виртуализованных сред выполнения объекта оценки (далее - ОО) в формате [QEMU](https://wiki.qemu.org/Main_Page), запуска данных сред под контролем Natch, анализа информации о движении помеченных данных, получаемой Natch от контролируемой виртуализованной среды.

## 1. Общие вопросы

### 1.1. Общие принципы работы Natch

Natch (Network Application Tainting Can Help) - это инструмент для определения поверхности атаки, основанный на полносистемном эмуляторе Qemu. Основная функция Natch - получение списка модулей (исполняемых файлов и динамических библиотек) и функций, используемых системой во время выполнения задачи. Natch представляет собой набор плагинов для эмулятора Qemu.

Общие принципы работы Natch, доступные плагины, команды управления Natch и их параметры представлены в веб-странице руководства, доступного в комплекте поставки (открыть браузером файл `index.html` в каталоге `/docs/manual/`).

В настоящий момент Natch поддерживает анализ только бинарного кода - таким образом анализ задействования кода интерпретируемых скриптов, а также "распространения" помеченных данных по коду интерпретируемых скриптов, возможен только в опосредованном виде - в формате анализа задействования нативных функций интерпретаторов, выполняющих указанные скрипты. 

#### 1.1.1. Комплект поставки

Комплект поставки Natch доступен в двух форматах:

- **основной** - защищенный бинарный дистрибутив, требующий наличие аппаратного ключа (персональный "черный" ключ, сетевой "красный" ключ или иные версии ключа) с лицензией c идентификатором "6":

    [Natch v.2.0](https://nextcloud.ispras.ru/index.php/s/natch_v.2.0) 

- **резервный** - .ova-образ Ubuntu 20 для VirtualBox с предустановленным защищенным Natch, необходимым ПО (pip3, vim) и доступом к VPN-серверу, раздающему лицензии:

    [Natch v.2.0](https://nextcloud.ispras.ru/index.php/s/learning)

Предыдущие версии защищенного бинарного дистрибутива Natch:

[Natch v.1.3.2](https://nextcloud.ispras.ru/index.php/s/natch_v.1.3.2) 

### 1.2. Общие принципы подготовки виртуализованной среды в формате QEMU

Процесс подготовки виртуализованной среды выполнения ОО в общем случае состоит из следующих последовательных шагов:
 - создание образа эмулируемой операционной системы в формате диска [qcow2](https://en.wikipedia.org/wiki/Qcow) на основе базового дистрибутива ОС. Формат qcow2 позволяет эффективно формировать снэпшоты состояния файловой системы в произвольный момент выполнения виртулизованной среды функционирования;
 - сборка дистрибутива ОО с требуемыми параметрами, в частности с генерацией и сохранением отладочных символов;
 - помещение собранного дистрибутива ОО в виртуализованную среду выполнения; - 
 - подготовка команд запуска QEMU, обеспечивающих эмуляцию аппаратной составляющей среды функционирования, загрузку и выполнение компонент Natch. 

Процесс подготовки виртуализованной среды выполнения ОО в значительной степени совпадает с процессом подготовки виртуализованной среды для анализа с помощью инструмента динамического анализа помеченных данных [Блесна](https://www.ispras.ru/technologies/blesna/) (разработчик - [ИСП РАН](https://www.ispras.ru/)), с точностью до подготовки команд запуска QEMU.

Процесс подготовки виртуализованной среды выполнения ОО рекомендуется выполнять в хостовой системе, допускающей запуск QEMU в режиме пользовательской виртуализации (ключ `-enable-kvm`) - это существенно ускорит процесс, скорость работы в режиме аппаратной виртуализации более чем на порядок превосходит работу в режиме полносистемной эмуляции. Проверить доступность данного режима в вашей хостовой системе (равно как и установить kvm-модули в вашу систему) можно опираясь на следующую [статью](https://phoenixnap.com/kb/ubuntu-install-kvm) с помощью команды:

```bash
sudo kvm-ok
```

при написании данной статьи в моей хостовой системе такая возможность отсутствовала, но менять конфигурации хостовой ВМ VirtualBox мне не хотелось:

```bash
[sudo] password for user: 
INFO: Your CPU does not support KVM extensions
KVM acceleration can NOT be used
```

При этом важно помнить, что собственно **запись трассы в любом случае необходимо выполнять без использования данного ключа**, поскольку запись осуществляется в режиме полносистемной эмуляции, собственно и позволяющей собрать полный лог действий процессора.

### 1.3. Пример подготовки виртуализованной среды в формате QEMU

#### 1.3.1. Подготовка хостовой системы

Рекомендации по подготовке хостовой системы приведены [здесь](https://gitlab.community.ispras.ru/trackers/natch/-/wikis/Natch-requirements#%D0%B0-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D0%BD%D1%8B%D0%B5-%D1%82%D1%80%D0%B5%D0%B1%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F-%D0%BA-%D1%85%D0%BE%D1%81%D1%82%D0%BE%D0%B2%D0%BE%D0%B9-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D0%B5) (*для получения доступа к репозиторию сообщества ознакомьтесь с информацией в описании телеграм-канала [Орг. вопросы::Доверенная разработка](https://t.me/sdl_community)*).

Подготовим Linux-based рабочую станцию (далее - хост), поддерживающую графический режим выполнения (QEMU демонстрирует вывод эмулируемой среды выполнения в отдельном графическом окне, следовательно нам необходим графический режим). Рабочая станция может быть реализована в формате виртуальной машины. Данное руководство описывает действия пользователя, работающего в виртуальной машине VirtualBox (4 ядра, 8 ГБ ОЗУ) с установленной ОС [Ubuntu20:04](https://releases.ubuntu.com/20.04/ubuntu-20.04.3-desktop-amd64.iso) (desktop-конфигурация, обновить пакеты при установке). 

Установим требуемое системное ПО, в т.ч. QEMU: 

```bash
sudo apt install  -y curl qemu-system gcc g++ 
```

*Подсказка: данная инсталляция требуется не для запуска Natch, но для создания образов ВМ на произвольном хосте. Natch содержит в своём составе требуемую для работы версию QEMU, поэтому если вы планируете создавать образ ВМ на том же хосте, на котором уже установили Natch, необходимость дополнительно устанавливать QEMU отсутствует*

Рекомендации по подготовке гостевой системы приведены [здесь](https://gitlab.community.ispras.ru/trackers/natch/-/wikis/Natch-requirements#%D0%B1-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D0%BD%D1%8B%D0%B5-%D1%82%D1%80%D0%B5%D0%B1%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F-%D0%BA-%D0%B3%D0%BE%D1%81%D1%82%D0%B5%D0%B2%D0%BE%D0%B9-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D0%B5-%D0%B2%D0%B8%D1%80%D1%82%D1%83%D0%B0%D0%BB%D1%8C%D0%BD%D0%BE%D0%B9-%D0%BC%D0%B0%D1%88%D0%B8%D0%BD%D0%B5).

Скачаем на хост выбранный базовый дистрибутив ОС. Поскольку виртуальные машины QEMU в режиме полносистемной эмуляции (анализ выполняется именно в таком режиме (параметр `--enable-kvm` в строке запуска QEMU должен отсутствовать), поскольку нам требуется полная эмуляция процессора и, частично, периферии для сбора максимально полной отладочной информации) значительно замедляют выполнение анализируемой среды функционирования, рекомендуется использовать минимальный образ - уменьшение числа установленных, стартующих при запуске служб, позволит значительно сократить нагрузку на процессор и сократить время, требуемое на запись и анализ трасс в режиме полносистемной эмуляции. В данном примере для ускорения процесс в качестве базового выбран легковесный образ Ubuntu - [lubuntu](https://lubuntu.net/downloads/) - скачаем его на хост, предварительно создав на хосте какой-нибудь каталог для дальнейших экспериментов:

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

Создадим скрипт запуска нашей ВМ `run.sh` - скрипт (равно как и дальнейшие скрипты в данном руководстве) достаточно объемный, поэтому желательно именно сохранять его в виде отдельного файла, допускающего удобное внесение изменений. Для тех, кто сталкивается с синтаксисом QEMU впервые, настоятельно рекомендуется ознакомиться с основными командами, подробно расписанными в официальной [документации QEMU](https://www.qemu.org/docs/master/system/invocation.html), а также в разделе руководства, посвященном QEMU.

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

после чего увидим знакомое нам графическое окно установки lubuntu. Выполним установку lubuntu - желательно выполнять её с минимальным набором параметров - для ускорения процесса установки и минимизации потенциального "шума" избыточных процессов и сетевых служб в записях сетевого трафика и трасс выполнения. 

*Подсказка: чтобы вывести курсор мыши из открытого графического окна ВМ QEMU нажмите Ctrl+Alt+G*

После завершения установки удалим из скрипта запуска `run.sh` указание подключения cdrom - для дальнейшей работы он нам не потребуется

```bash
#-cdrom lubuntu-18.04.3-desktop-amd64.iso
```
Наш образ среды функционирования готов к работе - в частности к установке в него пресобранного **с символами** прототипа объекта оценки.

#### 1.3.2. Сборка прототипа объекта оценки

Рекомендации по подготовке исполняемого кода приведены [здесь](https://gitlab.community.ispras.ru/trackers/natch/-/wikis/Natch-requirements#%D0%B2-%D0%BF%D0%BE%D0%B4%D0%B3%D0%BE%D1%82%D0%BE%D0%B2%D0%BA%D0%B0-%D0%BF%D0%BE%D0%B4%D0%BB%D0%B5%D0%B6%D0%B0%D1%89%D0%B5%D0%B9-%D0%B0%D0%BD%D0%B0%D0%BB%D0%B8%D0%B7%D1%83-%D0%B3%D0%BE%D1%81%D1%82%D0%B5%D0%B2%D0%BE%D0%B9-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D1%8B).

В общем случае к подлежащему анализу исполняемому коду выставляется два требования:

- для исполняемого кода должна быть представлена отладочная информация в формате символов в составе исполняемых файлов, отдельно прилагаемых символов или map-файлов. Предоставление символов непосредственно в составе исполняемых файлов является основной и рекомендуемой стратегий - начиная с версии Natch v1.3 инструмент умеет самостоятельно доставать информацию об отладочных символах из исполняемых файлов, собранных *как минимум* компиляторами gcc и clang с сохранением отладочной информации (ключ компилятора `-g`, также рекомендуется сборка без оптимизаций в режиме `-O0`). Начиная с версии Natch v2.1 будет внедрен функционал автоматической подгрузки символов для наиболее популярных сборок операционных систем;

- в случае, если процессы сборки и анализа будут выполняться в различных средах функционирования (как правило сборка осуществляется на отдельном высокопроизводительном сборочном сервере), требуется обеспечить совместимость версий разделяемых динамических библиотек, в первую очередь glibc, из состава среды функционирования.

В данном разделе в качестве прототипа объекта оценки рассмотрим популярную программу wget, сборку которой осуществим в хостовой системе (условная "сборочница") с последующим помещением собранного дистрибутива в виртуализированную гостевую среду lubuntu. 

Для выполнения [*классического*](https://thoughtbot.com/blog/the-magic-behind-configure-make-make-install) подготовительного скрипта `configure`, входящего в комплект поставки wget, генерирующего make-файл, потребуется установить ряд дополнительных зависимостей (скрипт выведет их наименования в случае неудачного завершения), в том числе для моей системы потребовалось:

```bash
sudo apt install -y gnutls-dev gnutls-bin curl
```

*Подсказка: поскольку я собираюсь собирать wget из исходников мне нужен комплект заголовочных файлов, доступный как раз в dev-версии пакета gnutls*

Скачаем исходные тексты wget с репозитория в файловую систему хоста (как и обозначено выше, сборку будем проводить именно на хосте):
```bash
curl -o wget-1.21.2.tar.gz  'https://ftp.gnu.org/gnu/wget/wget-1.21.2.tar.gz'
tar -xzf wget-1.21.2.tar.gz && cd wget-1.21.2
```

Успешное выполнение скрипта `configure` с определенными предустановками (ключи компилятора `-g -O0` ) позволит в т.ч. установить параметры компилятора, обеспечивающие сохранение информации об отладочных символах. 


**Важное замечание - следующие два подраздела оставлены в качестве пособия для специфических случаев, когда возможность сборки исполняемого файла из исходных текстов с произвольными параметрами отсутствует, либо явно требуется получение отдельных [map-файлов](https://stackoverflow.com/questions/22199844/what-are-gcc-linker-map-files-used-for).**

#### 1.3.3. *Генерация map-файлов средствами компилятора*

Выполним скрипт: 

```bash
CFLAGS='-g -O0 -Xlinker -Map=output.map' ./configure && \
make
```

и проверим, что map-файлы создались. 

```bash
find . -name *.map
./src/output.map
./output.map
```

Полученные map-файлы можно поместить на хосте в явно обозначенный Natch`у каталог, содержащий исполняемые файлы wget - тогда Natch будет опираться на данные map-файлы при символизации соответствующиех процессов.

#### 1.3.4. *Генерация map-файлов сторонними инструментами*

Получение map-файлов для исполняемого файла, собранного с отладочными символами, возможно с помощью сторонних инструментов. Это может быть актуально в тех случаях, когда сборочный конвейер недоступен, либо получение от сборочного конвейера map-файлов в поддерживаемом Natch формате невозможно (например, использование специфического компилятора/компоновщика). Сгенерируем map-файлы с использованием бесплатной версии дизассемблера [IDA](https://hex-rays.com/ida-free/) - необходимо скачать установочный комплект по указанной ссылке, возможно в систему придётся доустановить библиотеки Qt `apt install -y qt5-default`.

После установки IDA необходимо запустить её, открыть интересующий нас исполняемый файл (в нашем случае это `wget`)

![image](https://user-images.githubusercontent.com/46653985/152351389-7cd54129-c087-4dc3-8bf6-be500f8193c3.png)

пройти процедуру генерации map-файла

![image](https://user-images.githubusercontent.com/46653985/152351322-d92d16e0-5650-4c96-a61a-3abeb992b18e.png)


выбрав экспорт всей возможной информации

<img src="https://user-images.githubusercontent.com/46653985/152351245-d3a3510f-adba-4919-adfb-be081a6800e8.png" width="200" height="200">

после чего убедиться, что map-файл появился в файловой системе

```bash
ll src | grep .map
-rw-rw-r--  1 user user  564686 фев  3 16:11 wget.map
```

#### 1.3.5. Перенос прототипа объекта оценки в образ ВМ

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

#### 1.3.6. Тестирование виртуализированной среды функционирования ОО

Запускаем ВМ скриптом `run.sh` с учетом отключенного ранее cdrom, дожидаемся загрузки ОС ВМ, авторизуемся в ОС от имени учетной записи test, пробуем выполнить обращение к произвольному сетевому ресурсу с помощью собранной нами версии wget:

```bash
cd /wget-1.21.2 && sudo ./wget ispras.ru
```

В результате вы должны увидеть приблизительно следующую картину в графическом окне QEMU, свидетельствующую о том, что ОО корректно выполняется в среде функционирования и сетевая доступность для ВМ обеспечена:

![image](https://user-images.githubusercontent.com/46653985/151779302-c2d59bd1-aed1-45ce-8f00-2702380a3157.png)


## 2. Обучающие примеры 

### 2.1. Анализ образа системы, содержащего тестовые комплекты пресобранных исполняемых файлов

Для выполнения данного примера потребуется:
- рабочая станция под управлением ОС Linux (традиционно Ubuntu 20.04). Отдельная установка пакета **qemu-system** не требуется, нужная версия входит в дистрибутив Natch;
- актуальный [дистрибутив](#111-комплект-поставки) Natch;
- подготовленный разработчиком [тестовый набор](https://nextcloud.ispras.ru/index.php/s/testing_2.0), включающий в себя минимизированный образ гостевой операционной системы Debian (размер qcow2-образа около 1 ГБ), а также два комплекта бинарных файлов (Sample1_bins и Sample2_bins), собранных с символами, к которым дополнительно прилагаются map-файлы. 

*Сценарий использования тестового комплекта Sample1_bins*

Исполняемый файл test_sample читает помеченный файл (таковым при конфигурировании Natch нужно установить файл sample.txt в гостевой ОС), в первой строке которого записан адрес Google, выдергивает эту первую (уже помеченную в процессе чтения файла) строчку и вызывает исполняемый файл test_sample_2 - в качестве параметра используется эта строка. test_sample_2 "курлит гугл" в файл curl.txt. 

*Сценарий использования тестового комплекта Sample2_bins*

Процесс сервера redis-server следует запустить командой `redis-server --port 5555 --protected-mode no` (разумеется при конфигурировании Natch следует настроить проброс порта 5555, чтобы он был доступен из хостовой системы), после чего соединиться с ним из хостовой системы клиентской утилитой `redis-cli -h localhost -p 15555` (её можно поставить например так `sudo apt install redis-tools`) и выполнить какие-нибудь действия - например `SET b VeryBigValue`. 


#### 2.1.1. Получение образа и дистрибутива

В случае выполнении действий в подготовленной виртуальной машине, содержащей Natch, самостоятельные скачивание и установка бинарного комплекта не требуются.

В случае установки в формате бинарного комплекта следует скачать его и распаковать - команда для скачивания тестового комплекта с помощью curl выглядит так `curl -o materials.zip 'https://nextcloud.ispras.ru/index.php/s/testing_2.0/download'`. Состав комплекта бинарной поставки в облачном хранилище выглядит примерно так: 

<img src="https://user-images.githubusercontent.com/46653985/190381059-d3e5f651-eabb-48f1-9087-25dd5720ee24.png" width="400" height="175">

После скачивания дистрибутива и обучающих материалов их следует распаковать - традиционно (но не обязетельно, реальное размещение файлов тестовых материалов непринципиально и зависит от ваших предпочтений) после распаковки содержимое каталога будет выглядеть примерно так:

<img src="https://user-images.githubusercontent.com/46653985/190384425-8ae1d2d0-d07c-4d7b-ad9d-5174f71de50d.png" width="650" height="140">

В каталоге `libs` размещаются используемые Natch библиотеки (подключаются с использованием стандартного механизма [preload](https://www.baeldung.com/linux/ld_preload-trick-what-is#:~:text=The%20LD_PRELOAD%20trick%20is%20a,a%20collection%20of%20compiled%20functions.) при запуске qemu-system и иных qemu-процессов). В каталоге `qemu_plugins...` помещаются собственно исполняемые файлы Natch. Каталог `docs`, содержащий веб-страницу руководства, располагается внутри каталога `qemu_plugins...`.

Учётные записи пользователей гостевой ОС: `user / user` и `root / root`.


#### 2.1.2. Установка Natch и Snatch 

Для работы Natch следует установить python-библиотеки, обеспечивающие работоспособность скриптов:

```bash 
user@natch1:~/natch_quickstart$ pip3 install -r qemu_plugins_2004_tainting_x64_natch/bin/natch_scripts/requirements.txt
```

Для работы Snatch следует запустить установочный скрипт и дождаться его успешного выполнения (сообщения об ошибках сборки некоторых python-пакетов можно игнорировать при условии того, что скрипт в целом завершается успешно):

```bash
user@natch1:~/natch_quickstart/snatch$ ./snatch_setup.sh
```

#### 2.1.3. Настройка Natch для работы с тестовым образом ОС

Процесс настройки состоит из двух этапов - автоматизированно (обязательный) и ручного (дополнительный, при необходимости тонкой настройки). Предназначение файлов конфигурации и их параметров **подробно расписано в документации**):

##### 2.1.3.1. Автоматизированная настройка

Автоматизированная настройка выполняется интерактивным скриптом `natch_run.py`, выводимые которым вопросы и примеры ответов на которые приведём далее. Запуск скрипта (**не забываем про необходимость прелоада библиотек**):

```bash
user@natch1:~/natch_quickstart$ LD_LIBRARY_PATH=/home/user/natch_quickstart/libs/ ./qemu_plugins_2004_tainting_x64_natch/bin/natch_scripts/natch_run.py Natch_testing_materials/test_image_debian.qcow2 
Image: /home/user/natch_quickstart/Natch_testing_materials/test_image_debian.qcow2
OS: Linux
```

Вводим имя проекта - будет создан каталог с таким именем:

```bash
Enter path to directory for project (optional): test1
Directory for project files '/home/user/natch_quickstart/test1' was created
Directory for output files '/home/user/natch_quickstart/test1/output' was created
```

Сколько памяти выдать гостевой виртуальной машине:

```bash
Common options
Enter RAM size with suffix G or M (e.g. 4G or 256M): 4G
```

При желании можно указать произвольное имя для главного конфигурационного файла Natch:
```bash
Natch options
Enter name of config file (optional): 
Now we will trying to create overlay: overlay created
```

Если наш сценарий предполагает передачу помеченных данных со сети (далее мы рассматриваем в качестве основного как раз сценарий №2 - взаимодействие с redis-сервером, слушающим tcp-порт 5555), нам следует передавать пакеты помеченных данных в гостевую ОС извне её - **перехват пакетов, отправитель и получатель которых "находятся" внутри гостевой ОС (localhost <--> localhost), в настоящий момент работает не стабильно и не рекомендуется к использованию**. Указываем Natch, какой порт мы хотим опубликовать в хостовую ОС:

```bash
Network option
Do you want to use ports forwarding? [Y/n] Y
Write the ports you want separated by commas (e.g. 7777, 8888, etc) 5555
Your port for connecting outside: 15555
```

Далее нам нужно указать пути к каталогам на хосте, содержащим копии бинарных файлов, размещенных в гостевой ОС - это как раз те самые файлы (собранные с символами, или с отдельными map-файлами), которые мы получили в ходе выполнения пункта [1.3.2](#132-сборка-прототипа-объекта-оценки). Нам нужно увидеть численное подтверждение того, что все помещенные в каталог файлы найдены (в данном комплекте их 2, Natch ищет все ELF-файла и соответствующие им по названию map-файлы на всю глубину вложенности каталогов):

```bash
Modules part
Do you want to create module config? [Y/n] Y
Enter path to maps dir: Natch_testing_materials/Sample2_bins
Your config file 'module_config.cfg' for modules was created
ELF files found: 2
Map files found: 2
```

Финальная стадия - конфигурирование технических параметров Natch, требующая тестового запуска виртуальной машины. В ходе данного запуска выполняется получение информации о параметрах ядра и заполнение ini-файла (данный файл не следует корректировать вручную, если только вы не абсолютно уверены в том, зачем вы это делаете). Вы можете отказаться от данного шага, в случае если этот файл уже был ранее создан для данного образа гостевой виртуальной машины - тогда вам потребуется указать к нему путь - однако в большинстве случаев вы вероятно будете создавать таковые файлы с нуля:

```bash
If you have a config file 'task_config.ini' you can copy it to work directory and skip tuning. Do you want to make tuning? [Y/n] Y

Now will be launch tuning. Don't close emulator
Three...
Two..
One.
Go!
QEMU 6.2.0 monitor - type 'help' for more information
(qemu) 
6.2.0
(c) 2020-2022 ISP RAS

Reading Natch config file...
[Tasks] No such file '/home/user/natch_quickstart/test1/task_config.ini'. It will be created.
Now tuning will be launched.

Tuning started. Please wait a little...
Generating config file: /home/user/natch_quickstart/test1/task_config.ini
Trying to find 12 kernel-specific parameters
[01/12] Parameter - task_struct->pid            : Found
[02/12] Parameter - task_struct->comm           : Found
[03/12] Parameter - task_struct->parent         : Found
[04/12] Parameter - files_struct fields         : Found
[05/12] Parameter - vm_area_struct size         : Found
[06/12] Parameter - vm_area_struct->vm_start    : Found
[07/12] Parameter - vm_area_struct->vm_end      : Found
[08/12] Parameter - vm_area_struct->vm_flags    : Found
[09/12] Parameter - mm->map_count               : Found
[10/12] Parameter - mm_struct fields            : Found
[11/12] Parameter - task_struct->mm             : Found
[12/12] Parameter - task_struct->state          : Found

Tuning completed successfully! Now you can restart emulator and enjoy! :)
```

Отлично, автоматизированная настройка и создание базовых скриптов завершены успешно, всё готово к записи трассы, о чём Natch сообщил нам дополнительно:

```bash
File '/home/user/natch_quickstart/test1/natch_config_linux.cfg' created. You can edit it before using Natch.

After checking config file you can launch:
	just Natch with help 'run.sh'
	Natch in record mode with help 'run_record.sh'
	Natch in replay mode with help 'run_replay.sh'
	Qemu without Natch with help 'run_qemu.sh'
```

Обратите внимание на файл настроек `natch_config_linux.cfg` - именно его мы будем редактировать при необходимости выполнения ручной настройки, а также на файл `natch_log.log` - в нём логируются основные результаты работы подпрограмм, входящих в комплект поставки Natch.


##### 2.1.3.2. Дополнительная ручная настройка

Отредактируем сгенерированный основной конфигурационный файл  Natch `natch_config.cfg` в соответствии с рекомендациями . _Не забываем, что необходимо раскомментировать также названия секций в квадратных скобках, а не только сами параметры._. Раскомментируем следующие секции (подробнее об их предназначении см. документацию п. 2.1):

Логирование сетевых пакетов, поступающих из источников, указанных в секции `[Ports]`, в pcap-файл:
```bash
[NetLog]
on=true
log=packets
```

Пометка файла в гостевой ОС (пригодится для выполнения тестового сценария №1):
```bash
[TaintFile]
list=sample.txt
```

Сбор покрытия по базовым блокам для просмотра покрытия в IDA Pro с использованием плагина Lighthouse:
```bash
[Coverage]
file=coverage
taint=true
```

##### 2.1.4. Запись трассы

Выполним запись трассы с интересующим нас сценарием выполнения:

```bash
user@natch1:~/natch_quickstart/test1$ ./run_record.sh 
```

**И получим ожидаемую ошибку - забыли про прелоад :)** Выполним запуск по правилам:

```bash
user@natch1:~/natch_quickstart/test1$ LD_LIBRARY_PATH=../libs/ ./run_record.sh
```

Введём логин и пароль учетной записи пользователя - user:user и запустим redis-сервер:

![image](https://user-images.githubusercontent.com/46653985/190400669-38291176-5bdb-4c7e-8e33-b7332184cd97.png)

Тестово соединимся с ним из хостовой ОС чтобы убедиться, что система в комплексе работает как надо:

```bash
user@natch1:~/natch_quickstart$ redis-cli -h localhost -p 15555

localhost:15555> SELECT 0
OK
localhost:15555> SET a b
OK
localhost:15555> get a
"b"
localhost:15555> exit
```

Теперь всё готово к записи трассы. **Важный момент - трасса длинная, включает в себя в том числе этап загрузки ОС - но помеченные данные появятся в практически в самом конце, когда мы собственно обратимся к redis-серверу.** Соответственно для существенного сокращения времени на анализ (последующее выполнение ./run_replay.sh) нам желательно и рекомендуется сделать снэпшот в точке, максимально приближенной к точке начала поступления помеченных данных в системе. То есть сейчас, когда от порождения помеченных данных нас отделяет только повторное соединение с redis-сервером из хостовой ОС и повторная отправка в него тех или иных уже знакомых нам команд.

Нажмем `Ctrl (в моём случае левый) + Alt + G`, выйдем в монитор qemu (bash-терминал хостовой ОС в котором мы запустили run_record.sh) и выполним команду генерации снэпшота (займёт несколько секунд, в зависимости от размера образа и производительности компьютера в целом) - я назвал его `ready` - **команда savevm ready**:

![image](https://user-images.githubusercontent.com/46653985/190401680-ecef6dc6-dce3-437c-ac20-65eb1297a2fb.png)

После того, как снэпшот был сгенерирован, снова отправим какие-нибудь данные из хостовой ОС в redis-сервер, после чего закроем графическое окно и завершим выполнение qemu.

Поздравляю, трасса записана!

##### 2.1.5. Воспроизведение трассы

Перед воспроизведением трассы следует заменить значение параметра `$SNAPSHOT` в скрипте `run_replay.sh` при определении значения `rrshapshot=$SNAPSHOT` на имя нашего снэпшота, например используя редактор `vim`:

![image](https://user-images.githubusercontent.com/46653985/190402832-5b42bf9e-f690-4661-a79b-3aac7e429cf3.png)

Начнём воспроизведение трассы (приблизительно на порядок медленнее, чем базовой выполнение - вы моментально оцените пользу создания снэпшота):

```bash
user@natch1:~/natch_quickstart$ LD_LIBRARY_PATH=/home/user/natch_quickstart/libs/ ./test1/run_replay.sh
```

В ходе выполнения, в случае если вам требуются какие-то диагностики, имеющие отношение к конкретному моменту, в мониторе можно вводить различные команды, в частности команды плагинов Natch, указанные в п. 5 документации, например команду `show_tasks`, возвращающую дерево процессов в гостевой ОС, на момент её выполнения:

![image](https://user-images.githubusercontent.com/46653985/169652276-5fc5e8e5-6d91-4e84-b53e-333074d39f9c.png)

Через какое-то время выполнение сценария завершится, графическое окно закроется, и вы должны будете увидеть сообщение наподобие приведённого ниже, свидетельствующее о том, что интересующие нас модули гостевой ОС (в данном случае это один модуль - redis-сервер) были распознаны успешно, и следовательно мы получим в отчетах корректно символизированную информацию.

```bash
QEMU 6.2.0 monitor - type 'help' for more information
(qemu) 
6.2.0
(c) 2020-2022 ISP RAS

Reading Natch config file...
Task graph enabled
Taint enabled
Config is loaded.
Module binary log file /home/user/natch_quickstart/test1/output/log_m_b.log created successfully
Modules: started reading binaries
Modules: finished with 2 of 2 binaries for analysis
thread_monitor: identification method is set to a complex developed at isp approach
Started thread monitoring
Process events binary log file /home/user/natch_quickstart/test1/output/log_p_b.log created successfully
Tasks: config file is open.
Binary log file /home/user/natch_quickstart/test1/output/log_t_b.log created successfully
Detected module /home/user/natch_quickstart/Natch_testing_materials/Sample2_bins/redis-server execution
```

Если работа системы завершилась успешно, и вы не словили например `core dumped` (о чём стоит немедленно сообщить в [трекер](https://gitlab.community.ispras.ru/trackers/natch/-/issues) с приложением всех артефактов), можно переходить к собственно анализу трассы.

##### 2.1.6. Анализ трассы в ручном режиме

Основные виды диагностики, предоставляемые Natch, расписаны в п. 4 руководства. Следует ознакомиться со списками:
- задействованных модулей
- задействованных функций
- функций, вызывавших задействованные функции

а также:
- графом движения помеченных данных
- сетевым трафиком, доступным в pcap-файле `on.pcap`, а также в файле лога помеченных данных `tnetpackets.log` 

а также проанализировать:
- какие функции в наибольшей степени взаимодействовали с помеченными данными
- покрытие по базовым блокам функций, взаимодействовавших с помеченными данными

Анализ покрытия по базовым блокам выполняется с использованием IDA Pro (протестировано на версиях 7.0, 7.2), общий алгоритм действий описан в п. 4.7.2 документации. В ходе его выполнения может потребоваться ручное сопоставление модуля, для которого собрано покрытие, с модулем, загруженным в IDA. Наиболее явная причина - несовпадение имён исполняемого файла и файла, распознанного Natch. Пример такового несовпадения приведён на рисунке ниже:

![image](https://user-images.githubusercontent.com/46653985/169868133-325ba022-0cef-4d12-be6a-0390d3f22178.png)

После выполнения маппинга в представленном выше меню в ручном режиме мы увидим приблизительно следующие сведения о покрытии:

![image](https://user-images.githubusercontent.com/46653985/169868949-e46d60e6-a2b5-47a5-8801-e6857953a7b6.png)

Также при выборе функции можно увидеть покрытие непосредственно по ассемблерным инструкциям (голубой цвет):

![image](https://user-images.githubusercontent.com/46653985/169868764-24e49753-294a-4722-a71f-633e1f233063.png)

Демонстрация покрытия по декомпилированному коду в настоящий момент не поддерживается.

Также можно открыть и изучить записанный файл сетевого трафика `wireshark packets.pcap`:

![image](https://user-images.githubusercontent.com/46653985/190404652-dd6c7c9b-5a48-48dc-87db-e78bef7f2bf1.png)





