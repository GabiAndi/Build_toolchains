# Constuir toolchain cruzado para ARM Qt con sistema Linux

**Aclaración:** ***este documento se revisó por ultima vez el 10 de febrero del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Constuir toolchain cruzado para ARM Qt con sistema Linux](#constuir-toolchain-cruzado-para-arm-qt-con-sistema-linux)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Introducción](#introducción)
  - [Referencias](#referencias)
  - [Antes de comenzar](#antes-de-comenzar)
  - [Preparando la Raspberry Pi](#preparando-la-raspberry-pi)
    - [Instalación de Raspberry Pi OS ***(Host)***](#instalación-de-raspberry-pi-os-host)
    - [Generación de claves SSH para la Raspberry Pi ***(Host)***](#generación-de-claves-ssh-para-la-raspberry-pi-host)
    - [Configurar la Raspberry Pi ***(RPi)***](#configurar-la-raspberry-pi-rpi)
    - [Habilitar rsync con permisos sudo ***(RPi)***](#habilitar-rsync-con-permisos-sudo-rpi)
    - [Instalar los paquetes de desarrollo ***(RPi)***](#instalar-los-paquetes-de-desarrollo-rpi)
    - [Configurar IP estática para el puerto ethernet ***(RPi)***](#configurar-ip-estática-para-el-puerto-ethernet-rpi)
    - [Actualizamos la IP ***(Host)***](#actualizamos-la-ip-host)
    - [Crear copia de seguridad ***(Host)***](#crear-copia-de-seguridad-host)
  - [Preparando el sistema anfitrión](#preparando-el-sistema-anfitrión)
    - [Software necesario ***(Host)***](#software-necesario-host)
    - [Creamos mas variables bash ***(Host)***](#creamos-mas-variables-bash-host)
    - [Configuración de directorios de trabajo ***(Host)***](#configuración-de-directorios-de-trabajo-host)
    - [Obtener los binarios de Qt ***(Host)***](#obtener-los-binarios-de-qt-host)
    - [Instalación de un compilador cruzado](#instalación-de-un-compilador-cruzado)
    - [Sincronizar el sysroot de la Raspberry Pi ***(Host)***](#sincronizar-el-sysroot-de-la-raspberry-pi-host)
      - [Sincronización mediante el copiado de archivos (recomendado)](#sincronización-mediante-el-copiado-de-archivos-recomendado)
      - [Sincronización mediante el comando *rsync* (mas lento)](#sincronización-mediante-el-comando-rsync-mas-lento)
    - [Arreglar los enlaces simbólicos ***(Host)***](#arreglar-los-enlaces-simbólicos-host)
    - [Añadir el archivo ld.so.conf ***(Host)***](#añadir-el-archivo-ldsoconf-host)
    - [Compilar Qt ***(Host)***](#compilar-qt-host)
    - [Sincronizar los binarios construidos con la Raspberry Pi ***(Host)***](#sincronizar-los-binarios-construidos-con-la-raspberry-pi-host)
    - [Actualizar vinculador en Raspberry Pi ***(RPi)***](#actualizar-vinculador-en-raspberry-pi-rpi)
  - [Configuración de Qt Creator](#configuración-de-qt-creator)
    - [Agregar un nuevo dispositivo](#agregar-un-nuevo-dispositivo)
    - [Agregar un nuevo depurador](#agregar-un-nuevo-depurador)
    - [Agregar los compiladores cruzados](#agregar-los-compiladores-cruzados)
    - [Agregar la versión de Qt](#agregar-la-versión-de-qt)
    - [Agregar Kit](#agregar-kit)
    - [Configurar opciones de CMake](#configurar-opciones-de-cmake)
  - [Deploy de las aplicaciones en CMake](#deploy-de-las-aplicaciones-en-cmake)

## Introducción

Este tutorial busca guiarlo en la configuración de la Raspberry Pi y del sistema anfitrion, para lograr compilar programas Qt desde nuestra PC y probarlos en la Raspberry Pi. Para ello se utilizó la última versión de ***Raspberry Pi OS*** y un host con sistema con ***Linux Mint***.

## Referencias

Con la salida de Qt6, la documentación oficial y los post explicativos se actualizaron. Con esto se puede extraer información que sirve para realizar el proceso de configuración. Anteriormente con Qt5 las páginas de documentación y recursos para compilación no servian para nada, por suerte actualmente eso cambió.

A continuación pongo a disposición todas las páginas que leí y me ayudaron:

* [Crosscompile qt6 for raspberry pi 4.](https://stackoverflow.com/questions/66200034/crosscompile-qt6-for-raspberry-pi-4)
* [Qt 6 on the Raspberry Pi on eglfs.](https://bugfreeblog.duckdns.org/2021/09/qt-6-on-raspberry-pi-on-eglfs.html)
* [Configure an Embedded Linux Device.](https://doc-snapshots.qt.io/qt6-dev/configure-linux-device.html)
* [Proyecto Raspberry Pi Cross Compilers.](https://github.com/abhiTronix/raspberry-pi-cross-compilers)
* [Tutorial base de este documento.](https://github.com/abhiTronix/raspberry-pi-cross-compilers/blob/master/QT_build_instructions.md)
* [Página oficial de Raspberry Pi.](https://www.raspberrypi.org/)
* [Link de recursos de los compiladores cruzados.](https://sourceforge.net/projects/raspberry-pi-cross-compilers/files/)
* [Link de recursos de Qt.](https://download.qt.io/)

Le recomiendo que **lea a conciencia**. El copiar y pegar comandos como un loco lo llevará a que nada le funcione y se termine por frustrar. Así que tomece el tiempo y la calma necesaria para leer este post, le aseguro que aprenderá bastante.

## Antes de comenzar

Cuando vea el indicador de **(Host)**, significa que los pasos marcados en esa sección se realizan en la maquina anfitriona. Mientras que si ve **(RPi)** es en la Raspberry.

La compilación cruzada de Qt requiere que esté disponible una compilación de host de Qt. Durante la compilación, se invocan desde allí herramientas como moc, rcc, qmlcachegen, qsb y otras. Por ejemplo, si se realiza una compilación cruzada para arm en una máquina x64, primero debe estar disponible una compilación x64 local de la misma versión de Qt. La ruta a esta compilación de Qt se pasará a configure o cmake.

## Preparando la Raspberry Pi

### Instalación de Raspberry Pi OS ***(Host)***

Lo que primero debemos hacer es descargar la imagen del sistema de la [página oficial de Raspberry.](https://www.raspberrypi.org/software/operating-systems/)

Para compilaciones con interfaz gráfica de usuario es recomendable descargar la imagen completa con software recomendado. En el caso que solo desee crear aplicaciones de consola o gráficas sin un servidor de ventana, puede descargar una imagen lite del sistema.

En nuestro caso buscamos una implementación de Qt con Opengl ES2.

Para luego tener todo referenciado, creamos un par de variables de bash:

~~~TEXT
export TODAY=$(date +'%Y-%m-%d')

export RPI_VERSION="2"
export RPI_MODEL="b"
export RPI_NAME="rpi$RPI_VERSION$RPI_MODEL"

export RPI_USER="pi"

export RPI_IMG_NAME="2022-01-28-raspios-bullseye-armhf-lite"
export RPI_IMG_SHA256="f6e2a3e907789ac25b61f7acfcbf5708a6d224cf28ae12535a2dc1d76a62efbc"
~~~

Una vez tengamos el archivo .zip, nos vamos hacia la carpeta que lo contiene, lo verificamos y descomprimimos:

~~~TEXT
echo $RPI_IMG_SHA256 $RPI_IMG_NAME.zip && sha256sum $RPI_IMG_NAME.zip

unzip $RPI_IMG_NAME.zip
~~~

Insertamos la targeta SD en donde se instalará el sistema. Formateamos la SD y grabamos la imagen. Podemos utilizar cualquier utilidad de discos, en mi caso utilice la utilidad *dd*:

~~~TEXT
sudo dd if=$RPI_IMG_NAME.img of=/dev/mmcblk0 status=progress

sync
~~~

Para poder contar con SSH desde el primer arranque, debemos crear un archivo vacio en la partición */boot* con nombre *ssh* o *ssh.txt*:

~~~TEXT
sudo mkdir /media/$USER/raspi-boot
sudo mount /dev/mmcblk0p1 /media/$USER/raspi-boot

sudo touch /media/$USER/raspi-boot/ssh

sudo umount /media/$USER/raspi-boot
sudo rm -rf /media/$USER/raspi-boot
~~~

Ya puede desconectar su targeta SD y encender su Raspberry Pi.

### Generación de claves SSH para la Raspberry Pi ***(Host)***

Escanee la IP que tomo la Raspberry Pi:

~~~TEXT
nmap 192.168.1.2-254 -p 22
~~~

Ahora añada la siguiente variables bash:

~~~TEXT
export RPI_IP="192.168.1.75"
~~~

Puede generar un conjunto de llaves SSH y transferirla a la Raspberry Pi con los siguientes comandos:

~~~TEXT
ssh-keygen -t rsa -b 4096 -C $USER -f $HOME/.ssh/id_rsa_rpi$RPI_VERSION

ssh-copy-id -i $HOME/.ssh/id_rsa_rpi$RPI_VERSION $RPI_USER@$RPI_IP
~~~

Ingrese el usuario ***pi*** y la contraseña ***raspberry***. Luego ya puede conectarse a la Raspberry Pi con la llave generada:

~~~TEXT
ssh $RPI_USER@$RPI_IP
~~~

Esto le ahorrara tiempo para poder conectarse y para poder utilizar los comandos de ***rsync***. Ademas de que Qt pueda emparejarse correctamente.

### Configurar la Raspberry Pi ***(RPi)***

Lo primero que debe hacer, una vez se conecte a la Raspberry Pi, es configurarla. Para ello ejecute el comando:

~~~TEXT
sudo raspi-config
~~~

Realizamos la configuración de periféricos, wifi y demás. Cuando terminemos reiniciamos.

Como paso siguiente actualizamos todo:

~~~TEXT
sudo apt update
sudo apt upgrade -y
~~~

Luego de esto debe reiniciar la Raspberry Pi y ya la tendremos lista para comenzar a configurarla para nuestro setup.

### Habilitar rsync con permisos sudo ***(RPi)***

Más adelante en esta guía, usaremos el comando ***rsync*** para sincronizar archivos entre el Host y la Raspberry Pi. Para algunos de estos archivos, se requieren derechos de root. Puede hacer esto con un solo comando de terminal de la siguiente manera:

~~~TEXT
echo "$USER ALL=NOPASSWD:$(which rsync)" | sudo tee --append /etc/sudoers
~~~

### Instalar los paquetes de desarrollo ***(RPi)***

Procedemos a instalar los paquetes y las dependencias necesarias para compilar nuestro conjunto de herramientas, para ello instale los siguientes requerimientos según considere necesario.

**Programas para la Raspberry Pi:**

~~~TEXT
sudo apt install gdbserver screen mc htop rsync git
~~~

**Librerias para la Raspberry Pi:**

~~~TEXT
sudo apt install libncurses6 libncursesw6 libncurses-dev libgmp-dev libisl-dev libexpat1-dev
~~~

**Programas y utilidades extras:**

~~~TEXT
sudo apt install build-essential gdb python3 python3-dev python2 python2-dev doxygen openssl unzip wget texinfo texlive autoconf automake gettext gperf autogen guile-3.0 guile-3.0-dev flex patch diffutils ninja-build bison cmake cmake-data dh-exec diffstat graphviz meson
~~~

**Libreria de GPIO (WiringPi):**

~~~TEXT
mkdir temp

cd temp

wget https://project-downloads.drogon.net/wiringpi-latest.deb

sudo dpkg -i wiringpi-latest.deb

cd ..

rm -rf temp
~~~

Puedes comprobarlo con:

~~~TEXT
gpio readall
~~~

**Requerimientos de Qt para X11:**

~~~TEXT
sudo apt install libfontconfig1-dev libfreetype6-dev libx11-dev libx11-xcb-dev libxext-dev libxfixes-dev libxi-dev libxrender-dev libxcb1-dev libxcb-glx0-dev libxcb-keysyms1-dev libxcb-image0-dev libxcb-shm0-dev libxcb-icccm4-dev libxcb-sync-dev libxcb-xfixes0-dev libxcb-shape0-dev libxcb-randr0-dev libxcb-render-util0-dev libxcb-util-dev libxcb-xinerama0-dev libxcb-xkb-dev libxkbcommon-dev libxkbcommon-x11-dev xorg-dev libgtk-3-dev libudev-dev libinput-dev libts-dev libmtdev-dev libssl-dev libdbus-1-dev libglib2.0-dev libxcb-xv0-dev libwxgtk3.0-gtk3-dev '^libxcb.*-dev' '^libxkb.*-dev' autopoint debhelper dh-autoreconf dh-strip-nondeterminism dwz intltool-debian libarchive-zip-perl libdebhelper-perl libfile-stripnondeterminism-perl libsub-override-perl po-debconf xutils-dev quilt xvfb libcdt5 libcgraph6 libgts-0.7-5 libgvc6 libgvpr2 libjsoncpp24 liblab-gamut1 libpathplan4 librhash0 comerr-dev default-libmysqlclient-dev firebird-dev firebird3.0-common firebird3.0-common-doc freetds-common freetds-dev krb5-multidev libasound2-dev libct4 libcups2-dev libcupsimage2-dev libdouble-conversion-dev libfbclient2 libgssrpc4 libib-util libkadm5clnt-mit12 libkadm5srv-mit12 libkdb5-10 libkrb5-dev libmariadb-dev-compat libmd4c-dev libodbc1 libpq-dev libpq5 libproxy-dev libpulse-dev libpulse-mainloop-glib0 libsybdb5 libtommath1 libzstd-dev odbcinst odbcinst1debian2 pkg-kde-tools unixodbc-dev
~~~

**Requerimientos de Qt para OpenGL ES2:**

~~~TEXT
sudo apt install libglew-dev libglfw3-dev libgles2-mesa-dev libgbm-dev libdrm-dev kmscube
~~~

Podemos verificar la instalación. Ya que es necesario glesv2 (solo funciona en modo consola):

~~~TEXT
kmscube
~~~

**Requerimientos de Qt para formato de imagenes:**

~~~TEXT
sudo apt install libjpeg-dev libpng-dev
~~~

**Requerimientos de Qt para base de datos:**

~~~TEXT
sudo apt install libsqlite3-dev libmariadb-dev
~~~

**Requerimientos de Qt para multimedia:**

~~~TEXT
sudo apt install libtiff-dev libavcodec-dev libavformat-dev libswscale-dev libv4l-dev libxvidcore-dev libx264-dev
~~~

**Requerimientos de Qt para multimedia gstreamer:**

~~~TEXT
sudo apt install libgstreamer1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-doc gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-qt5 gstreamer1.0-pulseaudio libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
~~~

**Requerimientos de Qt para Audio:**

~~~TEXT
sudo apt install libopenal-data libsndio7.0 libopenal1 libopenal-dev pulseaudio
~~~

**Requerimientos de Qt para Bluetooth:**

~~~TEXT
sudo apt install bluez-tools libbluetooth-dev
~~~

### Configurar IP estática para el puerto ethernet ***(RPi)***

Ya que es mas fácil y rápido conectar la Raspberry Pi a nuestra computadora mediante cable Ethernet, definiremos una IP estática para que sea mas sencillo ubicarla.

Editamos el archivo **/etc/dhcpcd.conf**:

~~~TEXT
sudo nano /etc/dhcpcd.conf
~~~

Y en la parte final añadimos las siguientes lineas:

~~~TEXT
interface eth0
static ip_address=192.168.1.100/24
~~~

Reiniciamos la Raspberry Pi y listo, terminamos de realizar la configuración. Podemos probar actualizar una ultima vez y la apagamos.

### Actualizamos la IP ***(Host)***

Actualizamos la variable de bash por la que configuramos en la Raspberry Pi:

~~~TEXT
export RPI_IP="192.168.1.100"
~~~

### Crear copia de seguridad ***(Host)***

En este punto es recomendable guardar una copia del estado actual de la tarjeta SD, en el caso que una configuración o comando erróneo rompa la instalación, podremos volver hasta este punto muy fácilmente. Lo podemos hacer con *dd*:

~~~TEXT
sudo dd if=/dev/mmcblk0 of="rpi$RPI_VERSION-($TODAY).img" status=progress

sudo chown $USER:$USER "rpi$RPI_VERSION-($TODAY).img"
~~~

## Preparando el sistema anfitrión

### Software necesario ***(Host)***

Nos aseguramos de tener bien actualizado nuestro sistema, luego, deberemos instalar paquetes necesarios para la compilación de nuestro toolchain:

~~~TEXT
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential python3 python3-dev python2 python2-dev doxygen git openssl unzip wget libncurses6 libncursesw6 libncurses-dev rsync texinfo texlive autoconf automake gettext gperf autogen guile-3.0 flex patch diffutils libgmp-dev libisl-dev libexpat-dev llvm-11 clang-11 libclang-11-dev
~~~

### Creamos mas variables bash ***(Host)***

Vamos a generar un par de variables mas para evitar errores de tipeo:

~~~TEXT
export TARGET="arm-linux-gnueabihf"

export QT_VERSION_MAJOR=6
export QT_VERSION_MINOR=2
export QT_SUBVERSION=3
export QT_VERSION="$QT_VERSION_MAJOR.$QT_VERSION_MINOR.$QT_SUBVERSION"
export QT_HOST_PATH="/opt/qt/$QT_VERSION/gcc_64"

export WORK_DIR="$HOME/build-toolchain"
export SRC_DIR="$WORK_DIR/src"
export BUILD_DIR="$WORK_DIR/build"

export INSTALL_DIR="/opt/qtrpi2"
export SYSROOT="$INSTALL_DIR/sysroot"

export QT_INSTALL_DIR="$INSTALL_DIR/qt$QT_VERSION"
~~~

### Configuración de directorios de trabajo ***(Host)***

Puede utilizar los siguientes comandos para crear una carpeta y utilizarla como espacio de trabajo para crear archivos binarios de Qt:

~~~TEXT
mkdir -p $WORK_DIR $SRC_DIR $BUILD_DIR
sudo mkdir -p $INSTALL_DIR $SYSROOT $SYSROOT/usr $QT_INSTALL_DIR
sudo chown $USER:$USER -R $INSTALL_DIR
~~~

### Obtener los binarios de Qt ***(Host)***

Ahora, podemos descargar los archivos fuente más recientes para Qt. Ejecutamos el siguiente comando para descargar los archivos fuente:

~~~TEXT
cd $SRC_DIR

wget http://download.qt.io/archive/qt/$QT_VERSION_MAJOR.$QT_VERSION_MINOR/$QT_VERSION/single/qt-everywhere-src-$QT_VERSION.tar.xz
~~~

Una vez descargado el archivo, lo extraemos:

~~~TEXT
tar -xf qt-everywhere-src-$QT_VERSION.tar.xz
~~~

### Instalación de un compilador cruzado

Para compilar binarios de una arquitectura a otra necesitamos un compilador cruzado. Podemos descargarlo [o crear uno propio.](arm-linux.md) Luego procedemos a copiarlo en la carpeta de instalación:

~~~TEXT
export CROSS_COMPILER_NAME="cross-pi-2-gcc-10.2.0"

tar -xf $CROSS_COMPILER_NAME.tar.xz -C $INSTALL_DIR

export CROSS_COMPILER_DIR="$INSTALL_DIR/$CROSS_COMPILER_NAME"
~~~

### Sincronizar el sysroot de la Raspberry Pi ***(Host)***

Este paso es muy importante. Lo que vamos a hacer es copiar todo el contenido de las carpetas de la raíz de la Raspberry a nuestra computadora. Haremos esto para que compilador cruzado sepa las librerías con la cuales debe generar los binarios.

#### Sincronización mediante el copiado de archivos (recomendado)

Esta es la forma mas rápida de copiar los archivos, ya que se inserta la SD en el Host y se copian directamente los archivos sin necesidad de verse limitado por la velocidad de conexión:

~~~TEXT
sudo mkdir /media/$USER/raspi-root
sudo mount /dev/mmcblk0p2 /media/$USER/raspi-root

sudo cp -r /media/$USER/raspi-root/lib $SYSROOT
sudo cp -r /media/$USER/raspi-root/usr/include $SYSROOT/usr
sudo cp -r /media/$USER/raspi-root/usr/lib $SYSROOT/usr
sudo cp -r /media/$USER/raspi-root/usr/local $SYSROOT/usr
sudo cp -r /media/$USER/raspi-root/usr/share $SYSROOT/usr

sudo chown $USER:$USER -R $SYSROOT

sudo umount /media/$USER/raspi-root
sudo rm -rf /media/$USER/raspi-root
~~~

#### Sincronización mediante el comando *rsync* (mas lento)

Podemos sincronizar remotamente los archivos con los siguientes comandos a travez de rsync. El unico inconveniente es que demora mas que la alternativa anterior:

~~~TEXT
rsync -az --rsync-path="sudo rsync" $RPI_USER@$RPI_IP:/lib $SYSROOT
rsync -az --rsync-path="sudo rsync" $RPI_USER@$RPI_IP:/usr/include $SYSROOT/usr
rsync -az --rsync-path="sudo rsync" $RPI_USER@$RPI_IP:/usr/lib $SYSROOT/usr
rsync -az --rsync-path="sudo rsync" $RPI_USER@$RPI_IP:/usr/local $SYSROOT/usr
rsync -az --rsync-path="sudo rsync" $RPI_USER@$RPI_IP:/usr/share $SYSROOT/usr
~~~

### Arreglar los enlaces simbólicos ***(Host)***

Los archivos que copiamos en el paso anterior todavía tienen enlaces simbólicos que apuntan al sistema de archivos en la Raspberry Pi (direccionamiento absoluto). Necesitamos modificar esto para que se conviertan en enlaces relativos desde el nuevo directorio sysroot en la máquina host. Podemos hacer esto con un script de Python, que descargamos de la siguiente manera:

~~~TEXT
wget https://raw.githubusercontent.com/abhiTronix/rpi_rootfs/master/scripts/sysroot-relativelinks.py
~~~

Una vez que se descarga, solo necesita hacerlo ejecutable y correrlo usando los siguientes comandos:

~~~TEXT
chmod +x sysroot-relativelinks.py
./sysroot-relativelinks.py $SYSROOT
~~~

### Añadir el archivo ld.so.conf ***(Host)***

El problema realmente ocurre cuando ld invocado por GCC comienza a resolver las dependencias de la biblioteca. Tanto GCC como ld conocen las bibliotecas que contienen sysroot, sin embargo, es posible que a LD le falte un componente crítico: el archivo */etc/ld.so.conf*.

Cuando se enlazan las bibliotecas, puede ser que estas no se encuentren porque faltan rutas. Entonces ld busca el archivo */etc/ld.so.conf*, pero como este no existe, entonces la compilación falla.

La solución es crear el archivo en el sysroot:

~~~TEXT
mkdir $SYSROOT/etc

nano $SYSROOT/etc/ld.so.conf
~~~

Y añadimos las rutas en donde tenemos las bibliotecas:

~~~TEXT
/usr/local/lib/arm-linux-gnueabihf
/lib/arm-linux-gnueabihf
/usr/lib/arm-linux-gnueabihf
/usr/lib/arm-linux-gnueabihf/libfakeroot
/usr/local/lib
~~~

### Compilar Qt ***(Host)***

Creamos el archivo *toolchain.cmake* que hará que nuestra cadena de herramientas este configurada a nuestro gusto:

~~~TEXT
cd $INSTALL_DIR

nano toolchain.cmake
~~~

Recuerde que cada variable dependerá de las rutas de sus herramientas y carpetas de destino. Añada y modifique lo siguiente:

~~~TEXT
cmake_minimum_required(VERSION 3.18)

include_guard(GLOBAL)

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_SYSROOT $SYSROOT)
set(CROSS_COMPILER $CROSS_COMPILER_DIR/bin)

set(ENV{PKG_CONFIG_PATH} "")
set(ENV{PKG_CONFIG_LIBDIR} ${CMAKE_SYSROOT}/usr/lib/pkgconfig:${CMAKE_SYSROOT}/usr/share/pkgconfig)
set(ENV{PKG_CONFIG_SYSROOT_DIR} ${CMAKE_SYSROOT})

set(CMAKE_C_COMPILER ${CROSS_COMPILER}/$TARGET-gcc)
set(CMAKE_CXX_COMPILER ${CROSS_COMPILER}/$TARGET-g++)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
~~~

Ahora si podemos realizar la configuración del toolchain de Qt:

~~~TEXT
cd $BUILD_DIR

cmake \
-DQT_HOST_PATH=$QT_HOST_PATH \
-DCMAKE_TOOLCHAIN_FILE=$INSTALL_DIR/toolchain.cmake \
-DCMAKE_BUILD_TYPE=Release \
-DBUILD_SHARED_LIBS=ON \
-DCMAKE_INSTALL_PREFIX=$QT_INSTALL_DIR \
-DCMAKE_STAGING_PREFIX=$QT_INSTALL_DIR \
-DFEATURE_system_harfbuzz=OFF \
-DFEATURE_system_doubleconversion=OFF \
-DINPUT_opengl=es2 \
-DQT_QPA_DEFAULT_PLATFORM=EGLFS \
-DQT_QMAKE_DEVICE_OPTIONS=CROSS_COMPILE=$CROSS_COMPILER_DIR/bin/$TARGET- \
-DQT_QMAKE_TARGET_MKSPEC=devices/linux-rasp-pi2-g++ \
-DBUILD_qtwebengine=OFF \
-DBUILD_qtwayland=OFF \
-DQT_BUILD_TESTS=FALSE \
-DQT_BUILD_EXAMPLES=FALSE \
-G "Ninja" \
-S $SRC_DIR/qt-everywhere-src-$QT_VERSION \
-B .
~~~

Puede [ver la guía de referencia,](https://github.com/qt/qtbase/blob/dev/cmake/configure-cmake-mapping.md) para las opciones de configuración de CMake.

Si no se emitio ningun error, verifique que se habilitaron las caracteristicas que usted desea.

Si quiere volver a configurar tendra que borrar todo en el directorio:

~~~TEXT
rm -rf *
~~~

Si todo salio bien, ahora podrá construir los binarios de Qt. Este proceso demorará su tiempo:

~~~TEXT
cmake --build . --parallel
cmake --install .
~~~

### Sincronizar los binarios construidos con la Raspberry Pi ***(Host)***

Ahora que tenemos compilados los binarios de Qt, tenemos que pasarlos a la Raspberry Pi:

~~~TEXT
rsync -az --rsync-path="sudo rsync" $QT_INSTALL_DIR $RPI_USER@$RPI_IP:/usr/local
~~~

### Actualizar vinculador en Raspberry Pi ***(RPi)***

Ingrese el siguiente comando para actualizar el dispositivo permitiendo que el enlazador encuentre los nuevos archivos binarios Qt:

~~~TEXT
echo /usr/local/qt$QT_VERSION/lib | sudo tee /etc/ld.so.conf.d/qt$QT_VERSION.conf

sudo ldconfig
~~~

## Configuración de Qt Creator

### Agregar un nuevo dispositivo

Para comenzar a configurar el Qt Creator, vamos a añadir un nuevo dispositivo Linux Genérico que se va a conectar mediante SSH.

En Qt Creator abrimos la configuración ***Tools -> Options -> Devices***.

Seleccionamos en ***Add*** y vamos añadiendo los campos correspondientes. Si todo salio bien, al final Qt Creator se podrá conectar con la Raspberry Pi.

### Agregar un nuevo depurador

Nos vamos a ***Kits -> Debuggers*** y añadimos el GDB que dispone el compilador cruzado.

### Agregar los compiladores cruzados

Lo siguiente es ir a la pestaña ***Compilers*** y agregar un compilador de *GCC* *C* y otro de *C++*, correspondientes a nuestros compiladores cruzados.

### Agregar la versión de Qt

Vamos a la pestaña ***Qt versions*** y buscamos el ejecutable qmake generado.

### Agregar Kit

En la pestaña ***Kits*** tenemos que agregar toda la configuración que hicimos anteriormente. Verificando que no haya ningún inconveniente.

### Configurar opciones de CMake

Por último lo que debemos hacer en la pestaña de kits es irnos a editar las banderas por defecto de CMake, y añadir lo siguiente:

~~~TEXT
QT_HOST_PATH:PATH=%{Qt:QT_HOST_PREFIX}
CMAKE_SYSROOT:PATH=/opt/qtrpi2/sysroot
~~~

Y listo tenemos nuestra Raspberry Pi y su conjunto de herramientas ya instalado y listo para compilar codigo cruzado.

## Deploy de las aplicaciones en CMake

Para poder correr remotamente nuestra aplicación, debemos añadir la siguiente linea en el *CMakeLists.txt* de nuestro proyecto:

~~~TEXT
set(INSTALL_DESTDIR "/home/pi/${PROJECT_NAME}")

install(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION "${INSTALL_DESTDIR}"
    BUNDLE DESTINATION "${INSTALL_DESTDIR}"
    LIBRARY DESTINATION "${INSTALL_DESTDIR}"
)
~~~

Esto copiará en el destino los ejecutables que se creen.