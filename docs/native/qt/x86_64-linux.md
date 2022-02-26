# Constuir Toolchain nativo con Qt

**Aclaración:** ***este documento se revisó por ultima vez el 14 de febrero del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Constuir Toolchain nativo con Qt](#constuir-toolchain-nativo-con-qt)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Introducción](#introducción)
  - [Referencias](#referencias)
  - [Preparando el sistema](#preparando-el-sistema)
    - [Configuración de las dependencias](#configuración-de-las-dependencias)
    - [Instalar los paquetes de desarrollo](#instalar-los-paquetes-de-desarrollo)
    - [Variables de entorno y carpetas](#variables-de-entorno-y-carpetas)
    - [Obtener los binarios de Qt](#obtener-los-binarios-de-qt)
    - [Configurando Qt](#configurando-qt)
  - [Configuración de Qt Creator](#configuración-de-qt-creator)
    - [Agregar la versión de Qt](#agregar-la-versión-de-qt)
    - [Agregar Kit](#agregar-kit)

## Introducción

Este tutorial busca guiarlo en la configuración de un entorno de Qt para compilación nativa, ya que podemos caer en la necesidad de compilar Qt para Android (que necesita una version nativa de Qt en nuestro sistema), o compilar una versión estática o dinámica de nuestro proyecto.

Si su distro no esta basada en Debian o un derivado de este, seguramente los comandos para instalar dependencias y demas variarán un poco.

## Referencias

Luego de leer bastante los post de configuración de Qt, di con un video en YouTube que explica bastante bien lo que buscamos hacer:

* [Qt 6 - Build from source (Both Dynamic and Static).](https://www.youtube.com/watch?v=2qsR8Dw8uzA)
* [Building Qt Sources.](https://doc-snapshots.qt.io/qt6-dev/build-sources.html)

Le recomiendo que **lea a conciencia**. El copiar y pegar comandos como un loco lo llevará a que nada le funcione y se termine por frustrar. Así que tomece el tiempo y la calma necesaria para leer este post, le aseguro que aprenderá bastante.

## Preparando el sistema

### Configuración de las dependencias

Nos aseguramos de tener bien actualizado nuestro sistema, luego, deberemos instalar paquetes necesarios para la compilación de nuestro toolchain:

~~~TEXT
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential python3 python3-dev python-is-python3 python2 python2-dev doxygen git openssl unzip wget libncurses6 libncursesw6 libncurses-dev rsync texinfo texlive autoconf automake gettext gperf autogen guile-3.0 flex patch diffutils libgmp-dev libisl-dev libexpat-dev clang llvm cmake ninja-build meson graphviz diffstat dh-exec
~~~

### Instalar los paquetes de desarrollo

Procedemos a instalar los paquetes y las dependencias necesarias para compilar nuestro conjunto de herramientas, para ello instale los siguientes requerimientos según considere necesario.

**Programas:**

~~~TEXT
sudo apt install gdbserver screen mc htop rsync git
~~~

**Librerias:**

~~~TEXT
sudo apt install libncurses6 libncursesw6 libncurses-dev libgmp-dev libisl-dev libexpat1-dev
~~~

**Requerimientos de Qt para X11:**

~~~TEXT
sudo apt install libfontconfig1-dev libfreetype6-dev libx11-dev libx11-xcb-dev libxext-dev libxfixes-dev libxi-dev libxrender-dev libxcb1-dev libxcb-glx0-dev libxcb-keysyms1-dev libxcb-image0-dev libxcb-shm0-dev libxcb-icccm4-dev libxcb-sync-dev libxcb-xfixes0-dev libxcb-shape0-dev libxcb-randr0-dev libxcb-render-util0-dev libxcb-util-dev libxcb-xinerama0-dev libxcb-xkb-dev libxkbcommon-dev libxkbcommon-x11-dev xorg-dev libgtk-3-dev libudev-dev libinput-dev libts-dev libmtdev-dev libssl-dev libdbus-1-dev libglib2.0-dev libxcb-xv0-dev libwxgtk3.0-gtk3-dev '^libxcb.*-dev' '^libxkb.*-dev'
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

### Variables de entorno y carpetas

Vamos a declarar un par de variables de entorno que mas tarde utilizaremos:

~~~TEXT
export QT_VERSION_MAJOR=6
export QT_VERSION_MINOR=2
export QT_SUBVERSION=3
export QT_VERSION="$QT_VERSION_MAJOR.$QT_VERSION_MINOR.$QT_SUBVERSION"

export QT_BUILD_TYPE="static"

export WORK_DIR="$HOME/build-toolchain"
export SRC_DIR="$WORK_DIR/src"
export BUILD_DIR="$WORK_DIR/build"

export INSTALL_DIR="/opt/qt/$QT_VERSION/gcc_64_$QT_BUILD_TYPE"
~~~

Procedemos a crear las carpetas

~~~TEXT
mkdir -p $WORK_DIR $SRC_DIR $BUILD_DIR
sudo mkdir -p $INSTALL_DIR

sudo chown $USER:$USER $INSTALL_DIR -R 
~~~

### Obtener los binarios de Qt

Ahora, podemos descargar los archivos fuente más recientes para Qt. Ejecutamos el siguiente comando para descargar los archivos fuente:

~~~TEXT
cd $SRC_DIR

wget http://download.qt.io/archive/qt/$QT_VERSION_MAJOR.$QT_VERSION_MINOR/$QT_VERSION/single/qt-everywhere-src-$QT_VERSION.tar.xz
~~~

Una vez descargado el archivo, lo extraemos:

~~~TEXT
tar -xf qt-everywhere-src-$QT_VERSION.tar.xz
~~~

### Configurando Qt

Ahora si podemos realizar la configuración del toolchain de Qt:

~~~TEXT
cd $BUILD_DIR

cmake \
-DCMAKE_BUILD_TYPE=Release \
-DBUILD_SHARED_LIBS=OFF \
-DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
-DBUILD_qtwebengine=OFF \
-DBUILD_qtwayland=OFF \
-DQT_BUILD_TESTS=FALSE \
-DQT_BUILD_EXAMPLES=FALSE \
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

## Configuración de Qt Creator

### Agregar la versión de Qt

Vamos a la pestaña ***Qt versions*** y buscamos el ejecutable qmake generado.

### Agregar Kit

En la pestaña ***Kits*** tenemos que agregar toda la configuración que hicimos anteriormente. Verificando que no haya ningún inconveniente.
