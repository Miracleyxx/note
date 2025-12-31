## linux-x86

### 系统版本glibc

**2.17**

```
    2  mkdir record
    3  sudo apt install g++ gcc cmake git gdb
    4  apt install g++ gcc cmake git gdb
    5  sudo apt install g++ gcc cmake git gdb
    6  sudo su
    7  su -
    8  history
    9  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease
   10  make -j4
   11  ldd ./bin/libmrc.so 
   12  cd bin/
   13  ls
   14  cd ../
   15  make -j4
   16  sudo rm -rf ./*
   17  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease
   18  make -j4
   19  sudo rm -rf ./*
   20  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease
   21  make -j4
   22  nm -C /usr/local/lib/libmp3lame.a 
   23  nm -C /usr/local/lib/libmp3lame.a | grep lame_set_VBR
   24  objump -x /usr/local/lib/libmp3lame.a 
   25  sudo yum install objdump
   26  nm -C /usr/local/lib/libmp3lame.a 
   27  make -j4
   28  ls
   29  mkdir mp3_lib
   30  ./configure --preifx=./mp3_lib
   31  ./configure -preifx=./mp3_lib
   32  ./configure --help
   33  ./configure -prefix=./mp3_lib
   34  ./configure -prefix=mp3_lib
   35  pwd
   36  ./configure -prefix=/home/lyc/Desktop/ffmpeg/lame-3.100/mp3_lib
   37  make -j4 && make install
   38  ls
   39  cd ..
   40  ls
   41  mkdir ffmpeg-4.4.1
   42  mv ffmpeg-4.4.1.zip ffmpeg-4.4.1
   43  cd ffmpeg-4.4.1/
   44  unzip ffmpeg-4.4.1.zip 
   45  zip ffmpeg-4.4.1.zip 
   46  ls
   47  zip -f ffmpeg-4.4.1.zip 
   48  zip -F ffmpeg-4.4.1.zip 
   49  cd ffmpeg-4.4.1/
   50  mkdir ffmpeg_lib
   51  sudo chmod 777 -R ./*
   52  ./configure --prefix=ffmpeg_lib --enable-fpic --enable-libmp3lame
   53  ./configure --help | grep fpic
   54  ./configure --help | grep p
   55  ./configure --help | grep pic
   56  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame
   57  sudo apt install yasm
   58  sudo yum install yasm
   59  sudo yum install nasm
   60  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame
   61  sudo vim /etc/profile
   62  source /etc/profile
   63  sudo vim /etc/ld.so.conf.d/ffmpeg.conf
   64  ldconfig
   65  sudo ldconfig
   66  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame
   67  cat ffbuild/config.log 
   68  sudo vim /etc/profile
   69  ldconfig -p
   70  ldconfig -p | grep mp3
   71  cat ffbuild/config.log 
   72  pkg-config --libs -cflags mp3lame
   73  pkg-config --libs --cflags mp3lame
   74  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame
   75  ls /usr/local/lib | grep ame
   76  sudo rm -rf /usr/local/lib/libmp3lame.s*
   77  ls /usr/local/lib | grep ame
   78  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame
   79  make -j4 && make install
   80  cd ffmpeg_lib/
   81  ls
   82  cd lib/
   83  ls
   84  cd ..
   85  ffmpeg 
   86  cd ..
   87  cd ffmpeg_lib/
   88  ls
   89  cd bin/
   90  ls
   91  ./ffmpeg 
   92  ./ffmpeg -encoders
   93  ./ffmpeg -encoders | grep mp3
   94  ./ffmpeg -encoders | grep h264
   95  ./ffmpeg -encoders | grep h265
   96  ./ffmpeg -encoders | grep opus
   97  ./ffmpeg -encoders | grep aac
   98  ./ffmpeg -encoders | grep al
   99  ./ffmpeg -encoders | grep pam
  100  ./ffmpeg -encoders | grep pcm
  101  ./ffmpeg -encoders | grep al
  102  ./ffmpeg -encoders | grep pcm_a
  103  ./ffmpeg -encoders | grep G.711
  104  ldd ffmpeg 
  105  cd ..
  106  ls
  107  cd ..
  108  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame --disable-vaapi
  109  make -j4 && make install
  110  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame --disable-vaapi -disable-shared
  111  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame --disable-vaapi --disable-shared
  112  make -j4 && make install
  113  cd fftools/
  114  ls
  115  cd ..
  116  cd ffmpeg_lib/
  117  ls
  118  cd bin/
  119  ls
  120  ldd ffmpeg 
  121  nm -C ffmpeg 
  122  cd ../
  123  cd lib/
  124  ls
  125  um -C libavcodec.a 
  126  nm -C libavcodec.a 
  127  nm -C libavcodec.a | grep lame_get_encoder_delay
  128  nm -C libavcodec.a | grep lame_
  129  nm /usr/local/lib/libmp3lame.a | grep lame_
  130  nm /usr/local/lib/libmp3lame.a | grep lame_init
  131  nm /usr/local/lib/libmp3lame.a | grep lame_close
  132  nm /usr/local/lib/libmp3lame.a | grep lame_encode_
  133  nm /usr/local/lib/libmp3lame.a | grep lame_encode_buffer
  134  ls
  135  cd Desktop/
  136  ls
  137  mkdir ffmpeg
  138  pws
  139  pwd
  140  cd ..
  141  ls
  142  cd ..
  143  ./configure --enable-static
  144  make -j4 && make install
  145  sudo make -j4 && sudo make install
  146  ./configure --enable-static
  147  clear
  148  sudo make -j4 && sudo make install
  149  cd ..
  150  rm -rf lame-3.100
  151  ls
  152  cd lame-3.100/
  153  ./configure --enable-static
  154  ls /usr/local/lib
  155  rm /usr/local/lib
  156  rm -rf /usr/local/lib/*
  157  sudo rm -rf /usr/local/lib/*
  158  ls /usr/local/lib
  159  sudo make -j4 && sudo make install
  160  ls /usr/local/lib
  161  rm -rf /usr/local/lib/libmp3lame.s*
  162  sudo rm -rf /usr/local/lib/libmp3lame.s*
  163  sudo rm -rf /usr/local/lib/libmp3lame.la
  164  ls /usr/local/lib
  165  sudo rm -rf ./*
  166  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame --disable-vaapi
  167  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease ..
  168  make -j4
  169  ls /usr/local/lib
  170  export LD_LIBRARY_PATH=/usr/local/lib
  171  make -j4
  172  nm -C /usr/loacl/lib/libmp3lame.a
  173  nm -C /usr/local/lib/libmp3lame.a
  174  nm -C /usr/local/lib/libmp3lame.a |grep  lame_set_bWriteVbrTag
  175  cd ..
  176  ls
  177  sudo vim CMakeLists.txt 
  178  sudo vim ../media/CMakeLists.txt 
  179  ls
  180  cd build/
  181  sudo rm -rf ./*
  182  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease ..
  183  make -j4
  184  sudo rm -rf ./*
  185  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease ..
  186  make -j4
  187  cd bin/
  188  ls
  189  cd ..
  190  sudo vim ../media/CMakeLists.txt 
  191  sudo vim ../../media/CMakeLists.txt 
  192  sudo rm -rf ./*
  193  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease ..
  194  make -j4
  195  cd bin/
  196  ls
  197  ./libmrc.so 
  198  ldd libmrc.so 
  199  um -C | grep lame_set_VBR_quality
  200  nm -C | grep lame_set_VBR_quality
  201  nm -C libmrc.so | grep lame_set_VBR_quality
  202  cd ..
  203  sudo vim ../../media/CMakeLists.txt 
  204  sudo rm -rf ./*
  205  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease ..
  206  make -j4
  207  ldd demo
  208  ldd ./bin/demo 
  209  sudo rm -rf ./*
  210  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease ..
  211  make -j4
  212  sudo rm -rf ./*
  213  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease ..
  214  make -j4
  215  history
  216  ls
  217  cd ../
  218  ls
  219  cd lame-3.100/
  220  ./configure --enable-pic
  221  ./configure --help
  222  ./configure --help | grep pi
  223  ./configure --help --with-pic
  224  ./configure --with-pic
  225  make -j4 && make install
  226  sudo make -j4 && sudo make install
  227  ls
  228  ls /usr/local/lib/libmp3lame.
  229  ls /usr/local/lib/
  230  sudo rm /usr/local/lib/libmp3lame.s*
  231  sudo rm /usr/local/lib/libmp3lame.al
  232  sudo rm /usr/local/lib/libmp3lame.la
  233  ls /usr/local/lib/
  234  ./configure --help
  235  sudo make -j4 && sudo make install
  236  sudo rm /usr/local/lib/libmp3lame.s*
  237  sudo rm /usr/local/lib/libmp3lame.la
  238  ./configure --help
  239  ./configure --with-pic=yes enable-shared=no --enable-static=yes
  240  ./configure --with-pic=yes --enable-shared=no --enable-static=yes
  241  sudo make -j4 && sudo make install
  242  ls /usr/local/lib/
  243  sudo rm /usr/local/lib/libmp3lame.s*
  244  sudo rm /usr/local/lib/libmp3lame.la
  245  ./configure --with-pic=yes --enable-shared=no --enable-static=yes
  246  ./configure --with-pic=yes --enable-shared=no --enable-static=yes | grep shared
  247  ./configure --with-pic=yes --enable-shared=no --enable-static=yes | grep static
  248  sudo make -j4 && sudo make install
  249  ls /usr/local/lib/
  250  sudo rm /usr/local/lib/libmp3lame.s*
  251  sudo rm /usr/local/lib/libmp3lame.la
  252  ./configure --with-pic=yes --enable-shared=no --enable-static=yes | grep pic
  253  ./configure --with-pic=yes --enable-shared=no --enable-static=yes | grep ic
  254  ./configure --with-pic=yes --enable-shared=no --enable-static=yes
  255  ./configure --with-pic=yes --enable-shared=no --enable-static=yes | grep PIC
  256  ./configure --with-pic=no --enable-shared=no --enable-static=yes | grep PIC
  257  ./configure --with-pic=yes --enable-shared=no --enable-static=yes
  258  make clean
  259  sudo make -j4
  260  sudo rm /usr/local/lib/libmp3lame.s*
  261  ls /usr/local/lib.
  262  ls /usr/local/lib/
  263  make clean
  264  ls
  265  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame --disable-vaapi --disable-shared
  266  make clean
  267  ./configure --prefix=ffmpeg_lib --enable-pic --enable-libmp3lame --disable-vaapi --disable-shared
  268  git status
  269  git diff media/CMakeLists.txt
  270  git status
  271  git diff test/record.cpp
  272  git log
  273  git pull
  274  cd test/build/
  275  cmake ..
  276  history
  277  ../../../../cmake-3.28.0-linux-x86_64/bin/cmake .. -DCMAKE_BUILD_TYPE=Rlease ..
  278  make -j4
  279  ls
  280  cd bin/
  281  ls
  282  mkdir mp4
  283  ./demo 
  284  cd ..
  285  make -j4
  286  cd bin/
  287  ./demo 
  288  history
```

## linux-arm

### 系统版本glibc

**2.28**