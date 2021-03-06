# Жизнь без стандартной библиотеки

Основной reference по набору команд [преобразованный в
HTML](https://www.felixcloutier.com/x86/).


## Инструменты для сборки без стандартной библиотеки

### Инструменты GNU

При компоновке с опцией `-nostdlib` линковщик не
включает функцию `main`, и не связывает
программу со стандартной библиотекой языка Си.

Получаемый на выходе файл - обычный выполняемый файл в
формате ELF, который можно выполнить в операционной системе.

Размещение различных секций файла при компоновке можно
указать в специальном ld-файле (подробнее см. [LD:
Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html)),
который указывается опцией `-T имя_файла`.

Для того, чтобы при компоновке не включалась лишняя
информация о том, каким компилятором собрана программа,
исползуется опция линковщика `--build-id=none`.

Для выделения кода самой программы из ELF-файла можно
использовать утилиту `objcopy`.

### Ассемблер NASM

Ассемблер `nasm` использует хоть и похожий на Intel, но всё
же немного отличающийся по синтаксису язык. Этот ассемблер,
в отличии от GNU, поддерживает много выходных форматов, в
том числе flat-файлы, предназначенные для непосредственной
заливки программатором или загрузки в память.

## Взаимодействие с внешним миром в Linux

Операционная система Linux реализует системные вызовы через
прерывание с номером `0x80`. В регистре `eax` хранится
номер системного вызова, в регистрах `ebx`, `ecx`, `edx`,
`esi`, `edi` передаются аргументы, а возвращаемое значение
передается через `eax`.

Номера системных вызовов на x86 перечислены в файле
`/usr/include/asm/unistd_32.h`.

Некоторые полезные системные вызовы:
 * `exit` (`_exit` в Си-нотации) = `1` - выход из программы;
 * `read` = `3` - чтение из файлового дескриптора;
 * `write` = `4` - запись в файловый дескриптор;
 * `brk` (`sbrk` в Си-нотации) = `45` - перемещение границы
   сегмента данных программы.

Пример (вывод строки `Hello`):
```asm
    .text
    ......
    mov   eax, 4  // 4 - номер write
    mov   ebx, 1  // 1 - файловый дескриптор stdout
    mov   ecx, hello_ptr // указатель на hello
    mov   edx, 5  // количество байт в выводе
    int   0x80    // системный вызов Linux
    ......
    .data
hello:
    .string "Hello"
hello_ptr:
    .long   hello
```

## Взаимодействие с внешним миром через BIOS или в DOS

До момента загрузки операционной системы, обработка
ввода-вывода осуществляется с помощью подпрограмм,
предоставляемых BIOS (Basic Input Output System).

Разные подсистемам ("сервисам") соответствуют различные
номера прерываний. Например, прерывание `0x10`
предназначено для вывода на экран, а прерывание `0x09` - за
чтение с клавиатуры.

Некоторые операционные системы, например DOS, не запрещают
использование прерываний BIOS, а дополняют их своими
механизмами.

Подробное описание функций BIOS и DOS -
[здесь](http://www.codenet.ru/progr/dos/).

Отдельно стоит рассмотреть взаимодействие с выводом на
экран. Поскольку вывод через прерывание является хоть и
универсальным, но все же медленным способом, то лучше
использовать прямую запись в видеопамять VGA.

Видеопамять VGA в архитектуре x86 располагается в диапазоне
`0xA000...0xDFFFF` (256Кб начиная с 640Кб), и делится на
"окна", - области, назначение которых зависит от
используемого [режима
работы](https://wiki.osdev.org/VGA_Hardware#Memory_Layout_in_text_modes).

В стандартном текстовом видеорежиме, вывод символа в
позицию `(X, Y)` осуществляется записью двух байт по адресу
`0xB8000+Y*80*2+X*2`, где младший байт означает код
символа, а старший - цвет символа и фона.


## Стадии загрузки системы

### Включение компьютера

Сразу после запуска компьютера, управление передаётся
программе из ROM-памяти (часто именуемую BIOS, хотя это не
совсем корректно), задача которой - выполнить
диагностику системы, определить конфигурацию оборудования,
и загрузить программу-загрузчик с определенного диска, чтобы
передать ей управление.

Программа-загрузчик может располагаться:
 * в классической PC-системе - в первых 512 байтах диска;
 * в современных системах с EFI/UEFI - выделяется
   определенная область в Flash-памяти на системной плате,
   куда установщик операционной системы записывает свой
   загрузчик.

### Загрузка системы через MBR

Master Boot Record имеет размер 512 байт, и состоит из двух
частей: программы-загрузчика и первичной таблицы разделов
диска. Признаком того, что MBR имеет загрузчик, является
значение `0x55AA` в последних двух байтах. Размер первичной
таблицы разделов для PC - 64 байта, таким образом, для
загрузчика остается всего 446 байт (512-2-64).

Если загрузчик является достаточно сложным (например, GRUB
в графическом режиме со всякими красивостями и умной
командной строкой), то его делят на две части: в MBR и
частично - на разделе диска.

Первые 446 байт загружаются с диска в память по адресу
`0x7C00`, а область памяти от `0x0000` до `0x7C00`
считается зарезервированной под стек. При этом, процессор
x86 работает в 16-битном реальном режиме, со старинной
сегментной адресацией памяти. Пример программирования MBR -
[здесь](http://joebergeron.io/posts/post_two.html).

Задача загрузчика - это найти на диске файл с *ядром*
системы, загрузить его в память, и передать ему
управление. Примеры файлов ядра:
 * `C:\msdos.sys` - для DOS;
 * `C:\Windows\System32\ntoskrnl.exe` - для Windows;
 * `/boot/vmlinuz` - символическая ссылка на zlib-сжатый
   образ ядра в Linux.

Формат файла ядра - как правило, соответствует обычному
исполняемому файлу (PE для Windows или ELF для Linux), но
на него накладываются некоторые ограничения о размещении
данных внутри файла, и кроме того, этот файл не может иметь
зависимости от каких-либо библиотек.

### Загрузка ядра Linux, формат `multiboot`.

Загрузчик GRUB загружает ELF-файл с ядром, распаковывает
его при необходимости, и размещает по адресу, начиная с
1Мб.

Далее загрузчик ищет Magic-метку заголовка `multiboot` в первых
32К загруженного файла ядра, сразу после которой идет набор
флагов и контрольная сумма заголовка. После этого заголовка, в самом
файле следует 16К памяти под стек, а сразу после него -
начало программы, которую нужно выполнять.

Таким образом, при компиляции ядер, необходимо строго
указывать очерёдность различных секций, чтобы GRUB смог
запустить ядро.

Подробнее - [здесь](https://wiki.osdev.org/Bare_Bones).

### Запуск ядра

Ядро регистрирует вектор прерываний, выполняет дальнейшую
инициализацию оборудования, загружая при необходимости
различные драйверы устройств. Когда ядро полностью
загружено, то выполняется загрузка первой программы, которая
выполняется в режиме пользователя:
 * `C:\command.com` - для DOS;
 * `C:\Windows\System32\smms.exe` - для Windows;
 * `/boot/initrd` - для Linux.

### Дальнейшие стадии запуска

Процесс `initrd`, в зависимости от дистрибутива:
 * классический Unix-way: запускает набор shell-скриптов в
 одном из подкаталогов `/etc/init.d/rcX.d`, где `X` -
 уровень запуска по умолчанию, прописанный в файле
 `/etc/inittab`;
 * SystemD-way: запускает программу `systemd`, которая
   имеет свой набор конфигурационных файлов, по которым
   строит дерево зависимостей различных служб, и запускает их.
