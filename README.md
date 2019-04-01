### Что это?
##### В данном репозитории содержится работа на конкурс IT-planet по компетенции "Лучший Свободный Диплом".

### Название работы: "Серверная операционная система на базе CentOS"

Конечным результатом проекта является ISO образ, содержащий в себе операционную систему, которую можно скачать отсюда.

##### Как запустить?
Для тестирования необходим любой Linux дистрибутив или флеш накопитель.

###### Вариант №1: Демо запуск без перезагрузки
      Требования:
        1. Любой Linux дистрибутив
        2. Любая shell оболочка
        3. наличие утилиты chroot

      Шаг 0: Экспортируем переменную:
            export $OS=/mnt/os
      Шаг 1: Монтирование образа
        sudo mkdir /mnt/os
        sudo mount ~/Downloads/os.iso /mnt/os
      Шаг 2: Монтирование псевдофайловых систем:

            mknod -m 600 $OS/dev/console c 5 1
            mknod -m 666 $OS/dev/null c 1 3
            mount -v --bind /dev $OS/dev
            mount -vt devpts devpts $OS/dev/pts -o gid=5,mode=620
            mount -vt proc proc $OS/proc
            mount -vt sysfs sysfs $OS/sys
            mount -vt tmpfs tmpfs $OS/run
            if [ -h $OS/dev/shm ]; then
              mkdir -pv $OS/$(readlink $OS/dev/shm)
            fi

      Шаг 3: Вход в виртуальное окружение

      chroot "$OS" /usr/bin/env -i
        HOME=/root TERM="$TERM"
        PS1='(yurevich_os chroot) \u:\w\$ '
        PATH=/bin:/usr/bin:/sbin:/usr/sbin
        /bin/bash --login
      Шаг 4: Тестирование работоспособности(например, добавим пользователя)
         useradd -m ITplanet

###### Вариант №2: Загрузка с флеш носителя.
      Требования:
        1. бутабельный flesh накопитель
        2. режим загрузки "legacy"(хотя возможно работает и с EUFI)
      Шаг 1: Запись на флеш накопитель
         Если хостовая система - linux:
           dd if=OS.iso of=/dev/адрес_устройства ( например /dev/sda1)
        Если хостовая система - windows, то воспользуйтесь любой утилитой для записи на флеш накопитель(например "Universal linux installer" или deamon tools lite)
      Шаг 2: Загрузитесь с флеш накопителя

### Процесс разработки:
      Разработка велась на хостовой linux системе. Первой стадией разработки было создание виртуального окружения. Это необходимо для того, что бы не загрязнять домашнюю систему и исключить влияние софта с не правильными версиями на конечную систему. 

      Второй стадией была именно разработка операционной системы под виртуальным окружением. Были скомпелированы как жизненно-необходимые пакеты, так и пользовательский софт. Самым ответственным в этой стадии была компиляция gcc(вместе с GMP, MPFR, MPC), glibc, binutils, perl, ядра linux и initramfs. 

      Стадия 0: Подготовка хостовой системы.
        С помощью менеджера пакетов на хостовую систему были установлен следующие зависимости, необходимые для компиляции виртуального окружения(временной системы):
           1. Binutils-2.17
           2. Bison-2.3
           3. Bzip2-1.0.4
           4. Coreutils-6.9
           5. Diffutils-2.8.1
           6. Findutils-4.2.31
           7. Gawk-4.0.1
           8. GCC-4.7
           9. Glibc-2.11
           10. Gzip-1.3.12
           11. M4-1.4.10
           12. Make-3.81
           13. Patch-2.5.4
           14. Perl-5.8.8
           15. Sed-4.1.5
           16. Tar-1.22
           17. Texinfo-4.7
           18. Xz-5.0.0
      Стадия 1: Создание виртуального окружения
        Все пакеты для вирутального окружения компилировались примерно следующим образом, отличаясь лишь флагами (а их было очень много), передаваемыми при конфигурации make файла и компиляции:
           1. ./configure --prefix --flag2 --flag3 --flagN -- конфигурируем Makefile
           2. make --flag1 --flag2 --flagN -- компиляция в локальную папку
           3. make check --flag1 --flag2 --flagN -- проверка корректности установки
           4. make install -- установка глобально
        Например, что бы скомпилировать gcc, понадобилось сделать три прохода(с целью экономии времени и места я приведу всего один подход):
           Проход 1:
              шаг 0: распоковать gcc
              шаг 1. распаковать архивы с mpfr, gmp, mpc в папку с распакованным gcc
              шаг 2: изменить расположение динамического компановщика следующим костылём: 
            
                for file in gcc/config/{linux,i386/linux{,64}}.h
                do
                  cp -uv $file{,.orig}
                  sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' \
                  -e 's@/usr@/tools@g' $file.orig > $file
                  echo '
                  #undef STANDARD_STARTFILE_PREFIX_1
                  #undef STANDARD_STARTFILE_PREFIX_2
                  #define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
                  #define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
                  touch $file.orig
                done
              шаг 3: В файле gcc/config/i386/t-linux64 заменить пути к библиотекам на /lib64
              шаг 4: сконфигурировать make файл:
                ../configure                                       \
                    --target=$LFS_TGT                              \
                    --prefix=/tools                                \
                    --with-glibc-version=2.11                      \
                    --with-sysroot=$LFS                            \
                    --with-newlib                                  \
                    --without-headers                              \
                    --with-local-prefix=/tools                     \
                    --with-native-system-header-dir=/tools/include \
                    --disable-nls                                  \
                    --disable-shared                               \
                    --disable-multilib                             \
                    --disable-decimal-float                        \
                    --disable-threads                              \
                    --disable-libatomic                            \
                    --disable-libgomp                              \
                    --disable-libmpx                               \
                    --disable-libquadmath                          \
                    --disable-libssp                               \
                    --disable-libvtv                               \
                    --disable-libstdcxx                            \
                    --enable-languages=c,c++
                шаг 5: установка локально, а потом глобально:
                  make && make install
        Весь ниже приведённый софт устанавливался примерно по такой схеме, как и gcc выше. Вот что было установлено в нашу временную систему:
          Binutils-2.32(2 подхода)
          GCC-8.3.0(3 подхода)
          Linux headers 5.0
          Glibc-2.29
          Libstdc++ из пакета gcc
          Binutils-2.32
          Tcl-8.6.9
          Expect-5.45.4
          DejaGNU-1.6.2
          M4-1.4.18
          Ncurses-6.1
          Bash-5.0
          Bison-3.3.2
          Bzip2-1.0.6
          Coreutils-8.30
          Diffutils-3.7
          File-5.36
          Findutils-4.6.0
          Gawk-4.2.1
          Gettext-0.19.8.1
          Grep-3.3
          Gzip-1.10
          Make-4.2.1
          Patch-2.7.6
          Perl-5.28.1
          Python-3.7.2
          Sed-4.7
          Tar-1.32
          Texinfo-6.6
          Util-linux-2.33.1
          Xz-5.2.4
