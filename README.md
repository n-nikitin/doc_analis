# Natch Quickstart

*(актуально на момент 31.01.2022)*

## Введение

Данное руководство является кратким курсом по освоению основных принципов использования системы определения ширины и глубины поверхности атаки Natch (разработчик - [ИСП РАН](https://www.ispras.ru/)). В руководстве рассмотрены вопросы создания виртуализованных сред выполнения объекта оценки (далее - ОО) в формате [QEMU](https://wiki.qemu.org/Main_Page), запуска данных сред под контролем Natch, анализа информации о движении помеченных данных, получаемой Natch от контролируемой виртуализованной среды.

## 1. Общие вопросы

### 1.1. Общие принципы работы Natch

Natch (Network Application Tainting Can Help) - это инструмент для определения поверхности атаки, основанный на полносистемном эмуляторе Qemu. Основная функция Natch - получение списка модулей (исполняемых файлов и динамических библиотек) и функций, используемных системой во время выполнения задачи. Natch представляет собой набор плагинов для эмулятора Qemu.

Общие принципы работы Natch, доступные плагины, команды управления Natch и их параметры представлены в веб-странице руководства `Natch v.1.1 — QEMU documentation.html`, доступном в комплекте поставки.

### 1.2. Общие принципы подготовки виртуализированной среды в формате QEMU

Процесс подготовки виртуализированной среды выполнения ОО в общем случае состоит из следующих последовательных шагов:
 - создание образа эмулируемой операционной системы в формате диска [qcow2](https://en.wikipedia.org/wiki/Qcow) на основе базового дистрибутива ОС. Формат qcow2 позволяет эффективно формировать снэпшоты состояния файловой системы в произвольный момент выполнения виртулизованной среды функционирования;
 - сборка дистрибутива ОО с требуемыми параметрами, в частности с генерацией и сохранением отладочных символов;
 - помещение собранного дистрибутива ОО в виртуализированную среду выполнения;
 - подготовка команд запуска QEMU, обеспечивающих эмуляцию аппаратной составляющей среды функционирования, загрузку и выполнение компонент Natch. 

Процесс подготовки виртуализированной среды выполнения ОО в значительной степени сходен с процессом подготовки виртуализированной среды для анализа с помощью инструмента динамического анализа помеченных данных [Блесна](https://www.ispras.ru/technologies/blesna/) (разработчик - [ИСП РАН](https://www.ispras.ru/)), с точностью до подготовки команд запуска QEMU.

Процесс подготовки виртуализированной среды выполнения ОО рекомендуется выполнять в хостовой системе, допускающей запуск QEMU в режиме пользовательской виртуализации (ключ `-enable-kvm`) - это существенно ускорит процесс, скорость работы в режиме аппаратной виртуализации более чем на порядок превосходит работу в режиме полносистемной эмуляции. Проверить доступность данного режима в вашей хостовой системе (равно как и установить kvm-модули в вашу систему) можно опираясь на следующую [статью](https://phoenixnap.com/kb/ubuntu-install-kvm) с помощью команды:

```bash
sudo kvm-ok
```

при написании данной статьи в моей хостовой системе такая возможность отсутствовала, но менять конфигурации хостовй ВМ VirtualBox мне не хотелось

```bash
[sudo] password for user: 
INFO: Your CPU does not support KVM extensions
KVM acceleration can NOT be used
```

### 1.3. Пример подготовки виртуализированной среды в формате QEMU

#### Подготовка хостовой системы

Подготовим Linux-based рабочую станцию (далее - хост), поддерживающую графический режим выполнения (QEMU демонстрирует вывод эмулируемой среды выполнения в отдельном графическом окне, следовательно нам необходим графический режим). Рабочая станция может быть реализована в формате виртуальной машины. Данное руководство описывает действия пользователя, работающего в виртуальной машине VirtualBox (4 ядра, 8 ГБ ОЗУ) с установленной ОС Ubuntu20:04 (desktop-конфигурация). 

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





++++++++++++++++++++++++++++++++++++++++++++++++++



## 2. Анализ образа системы, содержащего пресобранную с символами версию программы wget

### 2.1 Подготовка стенда 

Подготовить Linux-based рабочую станцию, поддерживающую графический режим выполнения. Рабочая станция может быть реализована в формате виртуальной машины. Проверенной конфигурацией является Ubuntu20:04, 8 ядер, 8 ГБ ОЗУ. 

Установить на хост требуемое системное ПО, в т.ч. qemu: 
```bash
sudo apt install  -y curl qemu-system
```

Получить представление об общих принципах функционирования и командах управления эмулятора qemu - в [первоисточнике](https://qemu-project.gitlab.io/qemu/system/quickstart.html) или в [переводе](http://onreader.mdl.ru/KVMVirtualizationCookbook/content/Ch01.html).

Создать каталог ~/natch_quickstart (название выбрано произвольно), скачать комлпект поставки Natch в данный каталог:
```bas
cd  ~/natch_quickstart && \
curl -o Natch_documentation.pdf 'https://nextcloud.ispras.ru/s/raFWeX6B7XgYWDt/download?path=%2FNatch%20v.1.1&files=Natch_Documentation.pdf&downloadStartSecret=w0qrqfp8d9' && \
curl -o libs.zip 'https://nextcloud.ispras.ru/s/raFWeX6B7XgYWDt/download?path=%2FNatch%20v.1.1&files=libs_1804_natch_release_latest.zip&downloadStartSecret=shqvee1d2de' && \
curl -o plugins.zip 'https://nextcloud.ispras.ru/s/raFWeX6B7XgYWDt/download?path=%2FNatch%20v.1.1&files=qemu_plugins_1804_natch_release_latest.zip&downloadStartSecret=j0vq3mla9o8' && \
unzip libs.zip && \
unzip plugins.zip && \
rm -f libs.zip plugins.zip
```

Ознакомиться с документацией на Natch в файле `Natch_Documentation.pdf`, в частности получить представление:

- об основных принципах работы Natch
- об основных видах анализа и реализующих их плагинах
- о типовых командах Natch

Скачать [обучающий комплект](https://nextcloud.ispras.ru/s/o4w2mSqYAP4xFY3), включающий в себя в том числе qemu-образ, содержащий пресобранную с отладочными символами версию программы wget. Наличие символов позволит Natch формировать более читабельные отчеты, подставляя имена реальных функций анализируемой программы вместо адресов в памяти. Скачать образ можно командой: 

```bash
cd  ~/natch_quickstart && \
curl -o wget_image.zip https://nextcloud.ispras.ru/s/o4w2mSqYAP4xFY3/download && \
unzip wget_image.zip && \
rm -f wget_image.zip && \
mv 'For Dmitriy' wget_image # Временное название каталога на сервере надо будет заменить на что-то общеупотребительное
```

