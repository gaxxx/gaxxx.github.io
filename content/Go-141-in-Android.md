+++
title = 'Setup Go for Android'
date = 2015-02-09
[taxonomies]
tags = ["golang", "android"]
+++

Build Android App in Android Studio with Go(1.4.1) support.

<!-- more -->


## Preparation

1. Linux 64bits (Archlinux for me)
2. [NDK](https://developer.android.com/tools/sdk/ndk/index.html#download)
3. [golang crosscompile](github.com/davecheney/golang-crosscompile.git)
4. golang mobile tools
5. [golang jni maker](https://github.com/gaxxx/go_jni_maker)
6. android studio

## Install NDK

* Download & Install NDK


```
//for arch
yaourt android-ndk
```

* Make toolchains for ndk

```
//set ndk root dir
export NDK_ROOT=/opt/android-ndk
//build toolchains
cd $NDK_ROOT
./build/tools/make-standalone-toolchain.sh --platform=android-9 --install-dir=$NDK_ROOT --ndk-dir=$NDK_ROOT --toolchain=arm-linux-androideabi-4.9
```

## Golang runtime bootstrap (for android)

```
//get golang crosscompile
git clone https://github.com/davecheney/golang-crosscompile.git
//source build script
source crosscompile.bash
//set some variables
export GOARM=7
export CC_FOR_TARGET=$NDK_ROOT/bin/arm-linux-androideabi-gcc
//!bootstrap
go-crosscompile-build android/arm
```

## Get mobile tools

```
go get golang.org/x/mobile
```

## Get Jni-maker

```
git clone https://github.com/gaxxx/go_jni_maker
```

Final step
* create a golang library in $GOPATH

```
mkdir $GOPATH/src/testgojni
cat << 'EOF' > $GOPATH/src/testgojni/test.go
package testgojni
func Test() int {
    return 1
}
EOF
```

* create a gradle based android studio project

```
//copy & set jni maker
cp <go_jni_maker>  <android_studio_project>/app/
//set CC & GOLIB in make.bash (if necc)
CC=/opt/android-ndk/bin/arm-linux-androideabli-gcc
GOLIB=testgojni
```

* RUN make.bash
* insert init snippet in MainActivity

```
protected void onCreate(Bundle savedInstanceState) {
    .....
    Go.init(getApplicationContext());
    Testgojni.Test();
}
```
* Have fun
