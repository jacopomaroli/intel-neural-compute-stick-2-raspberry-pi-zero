# Compiling openvino and running Intel Neural Compute Stick 2 for raspberry pi zero

Here I'm going to write down some notes to help people like me which are willing to attempt to make use of Intel Neural Compute Stick 2 and Openvino on the raspberry pi zero.

## Why?
1. Raspberry pi zero is cheap, small and still very well supported.
2. It makes it very convenient adding vision to DIY robotics projects.
3. It features onboard wifi and camera connector so you don't have to add more hardware to play with computer vision (if not a more powerful co-processor)

## What's so difficult about that? Why do I need this document anyway?
Well if you're here it means you must have hit a brickwall at a certain point so you might already have a clue.  
Anyway the main reason is that, at the time this document has been redacted, Intel's official line is that only armv7 32 bit is officially supported (embedded architectures wise).  
It means that Raspberry pi zero (being armv6 32-bit) is cut out.

You'll find few references about cross compiling it for armv7 but that won't produce working binaries for armv6. Why is that?  
Because what is considered "armhf" (arm hard float) is not universally agreed.  
Raspberry encompasses in its definition of "armhf" both armv6 and armv7.  
Debian (hence all of its derivates such as Ubuntu Mint etc... except for rasbian in this case, of course) calls "armhf" only armv7.

## Why a document and not a github actions CI?
In one word: Time.  
Although if this gets enough traction/help/donations/a big series A round of funding for a dedicated startup I'll think about it. Maybe it'll grow until it gets merged in the main project.  
I think it's more likley though that Raspberry will drop an armv7 64-bit Raspberry pi zero w AI IoT 2.0 by then

## What's going to be the process?
We're going to compile everything:
1. qemu 4.1.0 - **Yes, we do absolutely need that.** The raspberry pi image will trigger `Unsupported syscall: 382` as documented here https://bugs.launchpad.net/qemu/+bug/1821006 if you'll just use the default qemu which comes with your Debian/Ubuntu/Mint.  
This will continuosly trip cmake's exception handling making impossible the compiling process.  
The version number is 4.1.0 because it comes from a tutorial I found. Please provide feedbacks if you succeed into using a more updated version.  
Also **cross compiling will not work** because of what aforementioned.
2. opencv 4.1.0 - No, the version doesn't have anything to do with the one of qemu. Is just because I found a few tutorials for this version but feel free to experiment with an updated one and provide feedbacks. Of course the one provided by your package manager is not going to be updated/personalized/flexible as the one you're going to compile
3. Openvino inference engine 2020.2 - Again, no specific reason for not picking the latest and greatest. I found a tutorial with this version and it was just one before the last one. Feedbacks are welcomed.

## 1. Building qemu
This part is mostly inspired by the following links:  
http://logan.tw/posts/2018/02/18/build-qemu-user-static-from-source-code/  
https://mathiashueber.com/manually-update-qemu-on-ubuntu-18-04/  
https://gist.github.com/julianxhokaxhiu/e7bc08c0702f7f13175f02eb68b8b447  
It's going to be minimal and featuring only an arm interpreter.  
For genaral purposes installation you might want to remove the last flag and build all the platforms.
```
wget https://download.qemu.org/qemu-4.1.0.tar.xz 
tar xvJf qemu-4.1.0.tar.xz 
cd qemu-4.1.0
mkdir build && cd build
../configure \
    --static \
    --disable-system \
    --enable-linux-user \
    --enable-attr \
    --target-list=arm-linux-user
make
sudo checkinstall # you should specify at least the package name (I put qemu) and the version (4.1.0)
```
## 2. Building openCV
Most of this comes from here https://solarianprogrammer.com/2019/08/07/cross-compile-opencv-raspberry-pi-zero-raspbian/  

```
sudo apt install debootstrap binfmt-support
```
binfmt is an utility which is able to recognize the fingerprint of an executable file.  
Once this is identified it selects an interpreter which is loaded as a kernel module.
This interpreter is used to allow us to load executables from architectures different than the host one.  
For this reason we're going to select the qemu-arm-static as an interpreter for the rasbian env.  
Let's start by cloning the base env
```
mkdir ~/raspbian
sudo debootstrap --no-check-gpg --foreign --arch=armhf buster ~/raspbian http://archive.raspbian.org/raspbian
```
If you now use `cat -v | head -n 10` or `od -c | head -n 10` you'll see the fingerprint of the executable we need to identify the architecture (this step is just to show the reasoning behind the next line)  
```
0000000 177   E   L   F 001 001 001  \0  \0  \0  \0  \0  \0  \0  \0  \0
0000020 002  \0   (  \0 001  \0  \0  \0   0   ; 001  \0   4  \0  \0  \0
0000040   p 244 001  \0  \0 004  \0 005   4  \0      \0  \t  \0   (  \0
0000060 034  \0 033  \0 001  \0  \0   p   H 224 001  \0   H 224 002  \0
0000100
```
We can now run:  
```
/proc/sys/fs/binfmt_misc/
sudo update-binfmts --install qemu-arm-static /usr/local/bin/qemu-arm --magic '\x7f\x45\x4c\x46\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00' --mask '\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff'
```
which will tell the kernel to use the interpreter found at the specified path `/usr/local/bin/qemu-arm` (**inside the chroot**) when the magic string is found.  
now we need to do the following
```
sudo cp /usr/local/bin/qemu-arm ~/raspbian/usr/local/bin/qemu-arm
sudo chroot ~/raspbian /debootstrap/debootstrap --second-stage
# from https://github.com/debian-pi/raspbian-ua-netinst/issues/314#issuecomment-159754610
mkdir -p -m 755 ~/raspbian/dev/pts
sudo mount -t proc proc ~/raspbian/proc/
sudo mount -t sysfs sys ~/raspbian/sys/
sudo mount -t devtmpfs -o mode=0755,nosuid devtmpfs ~/raspbian/dev
sudo mount -t devpts -o gid=5,mode=620 devpts ~/raspbian/dev/pts
sudo chroot ~/raspbian apt update
sudo chroot ~/raspbian apt upgrade
sudo chroot ~/raspbian
```
I confess I had a few hiccups during the above so if the last line spits out an error, keep investigating what's broken with binfmts. I kept getting 
```
cannot run command '/bin/bash' No such file or directory.
```
But it went away after I kept fiddling with the interpreters. Although please let me know if the above procedure works fine.  

here is a reference for some useful commands you can use **only in case you need troubleshooting**:  
```
update-binfmts --remove qemu-arm-static /usr/local/bin/qemu-arm
update-binfmts --remove qemu-arm-static /usr/lib/binfmt-support/run-detectors
update-binfmts --remove qemu-arm-static /usr/lib/binfmt-support/run-detectors
update-binfmts --display
update-binfmts --display qemu-arm-static
update-binfmts --enable qemu-arm-static
update-binfmts --install qemu-arm-static /usr/local/bin/qemu-arm
ls /proc/sys/fs/binfmt_misc/
echo ':qemu-arm-static:M::\x7fELF\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/local/bin/qemu-arm:' | sudo tee /proc/sys/fs/binfmt_misc/register
```
Some other useful commands:  
```
exit # get back to your host shell
sudo chroot ~/raspbian # log into the qemu shell

# run the following if you reboot
sudo mount -t proc proc ~/raspbian/proc/
sudo mount -t sysfs sys ~/raspbian/sys/
sudo mount -t devtmpfs -o mode=0755,nosuid devtmpfs ~/raspbian/dev
sudo mount -t devpts -o gid=5,mode=620 devpts ~/raspbian/dev/pts
```
Now you should finally have an environment where you can compile both openCV and inference-engine.  
Let's add some locale to make happy some packages we're going to install later (this should be needed only once)
```
export LANGUAGE=en_US.UTF-8
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
locale-gen en_US.UTF-8
dpkg-reconfigure locales
```
Install deps we're going to need during the building process
```
apt install build-essential gfortran libglib2.0-dev-bin python3-pip python-dev python-numpy python-cython python3-dev python3-numpy python3-cython libtiff-dev zlib1g-dev libjpeg-dev libpng-dev libavcodec-dev libavformat-dev libswscale-dev libv4l-dev libxvidcore-dev libx264-dev
```
run the following if you plan on building openCV with GUI support (This is what the other tutorial suggested, but I didn't and GUI support worked anyway)
```
apt install libgtk-3-dev libcanberra-gtk3-dev
```
download sources...
```
mkdir /opencv_all && cd /opencv_all
wget -O opencv.tar.gz https://github.com/opencv/opencv/archive/4.1.0.tar.gz
tar xf opencv.tar.gz
wget -O opencv_contrib.tar.gz https://github.com/opencv/opencv_contrib/archive/4.1.0.tar.gz
tar xf opencv_contrib.tar.gz
cd opencv-4.1.0
mkdir build && cd build
```
...configure it...
```
cmake -D CMAKE_BUILD_TYPE=RELEASE \
    -D CMAKE_INSTALL_PREFIX=/opt/opencv-4.1.0 \
    -D OPENCV_EXTRA_MODULES_PATH=/opencv_all/opencv_contrib-4.1.0/modules \
    -D OPENCV_ENABLE_NONFREE=ON \
    -D BUILD_TESTS=OFF \
    -D BUILD_DOCS=OFF \
    -D BUILD_OPENCV_PYTHON2=ON \
    -D BUILD_OPENCV_PYTHON3=ON \
    -D BUILD_EXAMPLES=OFF ..
```
Note: you can disable the GUI support by turning off `WITH_GTK` flag to the above snippet by adding this:
```
-D WITH_GTK=OFF \
```
...and build it.
```
make -j4
make install/strip
cd /opt/opencv-4.1.0/lib/python3.7/dist-packages/cv2/python-3.7/
ln -s cv2.cpython-37m-arm-linux-gnueabihf.so cv2.so
cd /opt/opencv-4.1.0/lib/python2.7/dist-packages/cv2/python-2.7/
ln -s cv2.cpython-27m-arm-linux-gnueabihf.so cv2.so
```

compress the library and copy it on the raspberry
```
cd /opt
tar -cjvf /opencv_all/opencv-4.1.0-pizero.tar.bz2 opencv-4.1.0
```

now on the raspberry let's install some dependencies
```
sudo apt install libgtk-3-dev libcanberra-gtk3-dev libtiff-dev zlib1g-dev libjpeg-dev libpng-dev libavcodec-dev libavformat-dev libswscale-dev libv4l-dev libxvidcore-dev libx264-dev libjasper-dev python3-numpy python-numpy
```
uncompress
```
tar xfv opencv-4.1.0-pizero.tar.bz2
sudo mv opencv-4.1.0 /opt
sudo ln -s /opt/opencv-4.1.0 /opt/opencv
sudo ln -s /opt/opencv/lib/python2.7/dist-packages/cv2 /usr/lib/python2.7/dist-packages/cv2
sudo ln -s /opt/opencv/lib/python3.7/dist-packages/cv2 /usr/lib/python3/dist-packages/cv2
```
load in default library search path
```
echo 'export LD_LIBRARY_PATH=/opt/opencv/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```
Done!
## 3. Building Openvino Inference Engine
Note: please refer to https://github.com/openvinotoolkit/openvino/blob/master/build-instruction.md and https://github.com/skhameneh/OpenVINO-ARM64 if you need a better insight in the following steps.

Go back to your building machine in the qemu environment.  
Let's start by downloading the sources:
```
mkdir /openvino_all && cd /openvino_all
wget https://github.com/openvinotoolkit/openvino/archive/2020.2.tar.gz
tar xf 2020.2.tar.gz
mv openvino-2020.2 openvino
cd openvino
mkdir build && cd build
```
We'll need to comment out the following two lines in the `/openvino_all/openvino/cmake/arm.toolchain.cmake` because otherwise we'll trigger a cross compile build
```
# set(CMAKE_SYSTEM_NAME Linux)
# set(CMAKE_SYSTEM_PROCESSOR armv7l)
```
Now you have 2 options. Long story short: If you compile it (like I did) leaving everything as it is now you'll get an error saying `opencv: undefined symbol: __atomic_fetch_add_8` both at compile time and at runtime https://github.com/piwheels/packages/issues/59 https://github.com/protocolbuffers/protobuf/commit/55ed1d427ccc0d200927746329ac9b811dee77b9.  

There are two possible fixes
1. (suboptimal) you enable turn `NGRAPH_UNIT_TEST_ENABLE=OFF` as I did later on and load your python executables by prefixing `LD_PRELOAD=/usr/lib/arm-linux-gnueabihf/libatomic.so.1`
2. (better - untested) you can try the version of protobuf which fixes the problem. IDK if this approach will work so let me know.
you'll need to change protobuf dependency from version 3.7.1 to version 3.12.2 here `ngraph/cmake/external_protobuf.cmake:47` https://github.com/openvinotoolkit/openvino/blob/master/ngraph/cmake/external_protobuf.cmake#L47 https://github.com/NervanaSystems/ngraph/blob/edc65ca0111f86a7e63a98f62cb17d153cc2535c/cmake/external_protobuf.cmake#L47  

It'll take a long time to compile so choose carefully.

We're now ready to compile. This will take a loooong time. It took almost half a day on my virtual machine.  
Maybe enabling qemu vtx support might help?
```
export OpenCV_DIR=/opt/opencv-4.1.0/lib/cmake/opencv4/
cmake -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE="/openvino_all/openvino/cmake/arm.toolchain.cmake" \
    -DTHREADS_PTHREAD_ARG="-pthread" \
    -DENABLE_SSE42=OFF \
    -DTHREADING=SEQ \
    -DENABLE_GNA=OFF .. \
    -DENABLE_PYTHON=ON \
    -DENABLE_OPENCV=OFF \
    -DCMAKE_CXX_FLAGS="-march=armv6" \
    -DNGRAPH_UNIT_TEST_ENABLE=OFF \
    -DENABLE_PYTHON=ON \
    -DPYTHON_EXECUTABLE=/usr/bin/python3.7 \
    -DPYTHON_LIBRARY=/usr/lib/arm-linux-gnueabihf/libpython3.7m.so \
    -DPYTHON_INCLUDE_DIR=/usr/include/python3.7 && make --jobs=$(nproc --all)
```

When it's finally done compress the library and copy it to the raspberry

```
cd /openvino_all/openvino/bin/armv7l
tar -cjvf /openvino_all/openvino-2020.2-pizero.tar.bz2 Release --transform s/Release/openvino-2020.2/
```
add the udev rules to allow the device to register itself properly when connected via USB
```
sudo tee -a /etc/udev/rules.d/97-myriad-usbboot.rules >/dev/null <<EOF
SUBSYSTEM=="usb", ATTRS{idProduct}=="2150", ATTRS{idVendor}=="03e7", GROUP="users", MODE="0666", ENV{ID_MM_DEVICE_IGNORE}="1"
SUBSYSTEM=="usb", ATTRS{idProduct}=="2485", ATTRS{idVendor}=="03e7", GROUP="users", MODE="0666", ENV{ID_MM_DEVICE_IGNORE}="1"
SUBSYSTEM=="usb", ATTRS{idProduct}=="f63b", ATTRS{idVendor}=="03e7", GROUP="users", MODE="0666", ENV{ID_MM_DEVICE_IGNORE}="1"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo ldconfig
```
symlink all the things to give you the freedom to select different versions of openvino and add the libraries to the library search path.
```
sudo ln -s /opt/openvino-2020.2 /opt/openvino
echo 'export LD_LIBRARY_PATH=/opt/openvino/lib:/opt/openvino/lib/python_api/python3.7/openvino/inference_engine:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

you can now plug in the Intel Neural Stick 2 and issue the following to check if everything is working

```
/opt/openvino/hello_query_device
```
which should give you back something like
```
Available devices:
        Device: MYRIAD
        Metrics:
                DEVICE_THERMAL : UNSUPPORTED TYPE
                RANGE_FOR_ASYNC_INFER_REQUESTS : { 3, 6, 1 }
                SUPPORTED_CONFIG_KEYS : [ CONFIG_FILE PERF_COUNT EXCLUSIVE_ASYNC_REQUESTS DEVICE_ID VPU_MYRIAD_PLATFORM VPU_IGNORE_IR_STATISTIC VPU_CUSTOM_LAYERS VPU_MYRIAD_FORCE_RESET VPU_PRINT_RECEIVE_TENSOR_TIME LOG_LEVEL VPU_HW_STAGES_OPTIMIZATION ]
                SUPPORTED_METRICS : [ DEVICE_THERMAL RANGE_FOR_ASYNC_INFER_REQUESTS SUPPORTED_CONFIG_KEYS SUPPORTED_METRICS OPTIMIZATION_CAPABILITIES FULL_DEVICE_NAME AVAILABLE_DEVICES ]
                OPTIMIZATION_CAPABILITIES : [ FP16 ]
                FULL_DEVICE_NAME : Intel Movidius Myriad X VPU
        Default values for device configuration keys:
                CONFIG_FILE : ""
                PERF_COUNT : OFF
                EXCLUSIVE_ASYNC_REQUESTS : OFF
                DEVICE_ID : ""
                VPU_MYRIAD_PLATFORM : ""
                VPU_IGNORE_IR_STATISTIC : OFF
                VPU_CUSTOM_LAYERS : ""
                VPU_MYRIAD_FORCE_RESET : OFF
                VPU_PRINT_RECEIVE_TENSOR_TIME : OFF
                LOG_LEVEL : LOG_NONE
                VPU_HW_STAGES_OPTIMIZATION : ON
```

At this point we can now checkout the openzoo repo which contains the reference to download some pre trained models

```
cd /opt
sudo git clone https://github.com/opencv/open_model_zoo.git
cd open_model_zoo
sudo git checkout 2020.2
python3 -mpip install --user -r /opt/open_model_zoo/tools/downloader/requirements.in
sudo mkdir /opt/openzoo_models
sudo chown $(id -u):$(id -g) /opt/openzoo_models
/opt/open_model_zoo/tools/downloader/downloader.py --name face-detection-adas-0001 --output_dir /opt/openzoo_models
```

We can now run an example of face recognition
```
mkdir ~/hello_vino
cd ~/hello_vino
wget -O face.jpg https://upload.wikimedia.org/wikipedia/commons/thumb/a/a9/Mona_Lisa_detail_face.jpg/187px-Mona_Lisa_detail_face.jpg
/opt/openvino/object_detection_sample_ssd_c -i /home/pi/hello_vino/face.jpg -m  /opt/openzoo_models/intel/face-detection-adas-0001/FP16/face-detection-adas-0001.xml -d MYRIAD
```
this should create a file called out_0.bmp in the `~/hello_vino` folder with a square around the face of the subject

let's try and do the same with python
```
wget https://raw.githubusercontent.com/openvinotoolkit/openvino/2020.2/inference-engine/ie_bridges/python/sample/object_detection_sample_ssd/object_detection_sample_ssd.py
LD_PRELOAD=/usr/lib/arm-linux-gnueabihf/libatomic.so.1 PYTHONPATH=/opt/openvino/lib/python_api/python3.7 python3 /home/pi/hello_vino/object_detection_sample_ssd.py -i /home/pi/hello_vino/face.jpg -m /opt/openzoo_models/intel/face-detection-adas-0001/FP16/face-detection-adas-0001.xml -d MYRIAD
```
this should create another out0.bmp (no underscore this time) in the same folder as above.

If you reached this far well done, feel free to follow the multiple tutorials and how-to's which Intel and openvino made available.
