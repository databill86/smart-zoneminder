# tpu-servers
This folder contains code and collateral for running the object detection and face recognition or person classification servers using the Google edge TensorFlow Processing Unit (TPU). The Alarm Uploader can be configured to use these servers instead of GPU-based ones normally co-resident with the machine running ZoneMinder. 

The TPU-based server, [detect_servers_tpu.py](./detect_servers_tpu.py), runs [TPU-based](https://cloud.google.com/edge-tpu/) TensorFlow Lite inference engines using the [Google Coral](https://coral.withgoogle.com/) Python APIs and employs [zerorpc](http://www.zerorpc.io/) to communicate with the Alarm Uploader. One of the benefits of using zerorpc is that the servers can easily be run on another machine, apart from the machine running ZoneMinder (in this case a [Coral Dev Board](https://coral.withgoogle.com/products/dev-board/)).

Images are sent to the object detection as an array of strings, in the following form.
```javascript
['path_to_image_1','path_to_image_2','path_to_image_n']
```

Object detection results are returned as json. An example is shown below.
```json
[ { "image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00506-capture.jpg",
    "labels": 
     [ { "name": "person",
         "id": 0,
         "score": 0.98046875,
         "box": 
          { "xmin": 898.4868621826172,
            "xmax": 1328.2035827636719,
            "ymax": 944.9342751502991,
            "ymin": 288.86380434036255 } } ] },
  { "image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00509-capture.jpg",
    "labels": 
     [ { "name": "person",
         "id": 0,
         "score": 0.83984375,
         "box": 
          { "xmin": 1090.408058166504,
            "xmax": 1447.4291610717773,
            "ymax": 846.3531160354614,
            "ymin": 290.5584239959717 } } ] },
  { "image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00515-capture.jpg",
    "labels": [] } ]
```

The object detection results then in turn can be sent to the face detector/recognizer or person classifier. An example of the face or person recognition results returned is shown below.
```json
[ {"image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00506-capture.jpg",
    "labels": 
     [ { "box": 
          { "xmin": 898.4868621826172,
            "xmax": 1328.2035827636719,
            "ymax": 944.9342751502991,
            "ymin": 288.86380434036255 },
         "name": "person",
         "face": "lindo_st_angel",
         "faceProba": 0.88145875,
         "score": 0.98046875,
         "id": 0 } ] },
  { "image": "/nvr/zoneminder/events/PlayroomDoor/19/04/04/04/30/00/00509-capture.jpg",
    "labels": 
     [ { "box": 
          { "xmin": 1090.408058166504,
            "xmax": 1447.4291610717773,
            "ymax": 846.3531160354614,
            "ymin": 290.5584239959717 },
         "name": "person",
         "face": null,
         "faceProba": null,
         "score": 0.83984375,
         "id": 0 } ] } ]
```

# Installation
1. Using the [Get Started Guide](https://coral.withgoogle.com/tutorials/devboard/), flash the Dev Board with the latest software image from Google and [install](https://www.tensorflow.org/lite/guide/python) the TensorFlow Lite interpreter.

2. The Dev Board has a modest 8GB on-board eMMC. You need to insert a MicroSD card (at least 32 GB) into the Dev Board to have enough space to install the software in the next steps. The SD card should be auto-mounted so on power-up and reboots the board can operate unattended. I mounted the SD card at ```/media/mendel```. My corresponding ```/etc/fstab``` entry for the SD card is shown below.  

```bash
#/dev/mmcblk1 which is the sd card
UUID=ff2b8c97-7882-4967-bc94-e41ed07f3b83 /media/mendel ext4 defaults 0 2
```

3. Create a swap file.
```bash
$ cd /media/mendel

# Create a swapfile else you'll run out of memory compiling.
$ sudo mkdir swapfile
# Now let's increase the size of swap file.
$ sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 oflag=append conv=notrunc
# Setup the file as a "swap file".
$ sudo mkswap /swapfile
# Enable swapping.
$ sudo swapon /swapfile
```

4. Install zerorpc.
```bash
# Update repo.
$ sudo apt-get update

# Install dependencies if needed. 
$ sudo apt install python3-dev libffi-dev

$ pip3 install zerorpc

# Test...
$ python3
>>> import zerorpc
>>> 
```

5. Install OpenCV.

NB: The Coral's main CPU is a Quad-core Cortex-A53 which uses an Armv8 microarchitecture and supports single-precision (32-bit, aka AArch32) and double-precision (64-bit, aka AArch64) floating-point data types and arithmetic as defined by the IEEE 754 floating-point standard. OpenCV can use [SIMD (NEON)](https://developer.arm.com/architectures/instruction-sets/simd-isas/neon) instructions to accelerate its computations which is enabled by the cmake options as shown below. For more information about floating point operations from Arm, see [Floating Point](https://developer.arm.com/architectures/instruction-sets/floating-point).

```bash
# Update repo.
$ sudo apt-get update

# Install basic dependencies.
$ sudo apt install python3-dev python3-pip python3-numpy \
build-essential cmake git libgtk2.0-dev pkg-config \
libavcodec-dev libavformat-dev libswscale-dev libtbb2 libtbb-dev \
libjpeg-dev libpng-dev libtiff-dev libdc1394-22-dev protobuf-compiler \
libgflags-dev libgoogle-glog-dev libblas-dev libhdf5-serial-dev \
liblmdb-dev libleveldb-dev liblapack-dev libsnappy-dev libprotobuf-dev \
libopenblas-dev libgtk2.0-dev libboost-dev libboost-all-dev \
libeigen3-dev libatlas-base-dev libne10-10 libne10-dev liblapacke-dev

# Install neon SIMD acceleration dependencies.
$ sudo apt install libneon27-dev libneon27-gnutls-dev

# Download source.
$ cd /media/mendel
$ wget -O opencv.zip https://github.com/opencv/opencv/archive/3.4.5.zip
$ unzip opencv.zip
$ wget -O opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/3.4.5.zip
$ unzip opencv_contrib.zip

# Configure OpenCV using cmake.
$ cd /media/mendel/opencv-3.4.5
$ mkdir build
$ cd build
# NB VFPv3 is not used in Armv8-A (AArch66)...don't set -DENABLE_VFPV3=ON.
$ cmake -D CMAKE_BUILD_TYPE=RELEASE -D ENABLE_NEON=ON -D ENABLE_TBB=ON \
-D ENABLE_IPP=ON -D WITH_OPENMP=ON -D WITH_CSTRIPES=OFF -D WITH_OPENCL=ON \
-D BUILD_TESTS=OFF -D INSTALL_PYTHON_EXAMPLES=OFF D BUILD_EXAMPLES=OFF \
-D CMAKE_INSTALL_PREFIX=/usr/local \
-D OPENCV_EXTRA_MODULES_PATH=/media/mendel/opencv_contrib-3.4.5/modules/ ..

# Compile and install. This takes a while...cross compile if impatient
$ make
$ sudo make install

# Rename binding.
$ cd /usr/local/lib/python3.5/dist-packages/cv2/python-3.5
$ sudo mv cv2.cpython-35m-aarch64-linux-gnu.so cv2.so

# Test...
$ python3
>>> import cv2
>>> cv2.__version__
'3.4.5'
>>>
```

6. Install scikit-learn.
```bash
# Install
$ pip3 install scikit-learn

# Test...
$ python3
>>> import sklearn
>>> sklearn.__version__
'0.20.3'
>>>
```

7. Install XGBoost.
```bash
# Install - see https://xgboost.readthedocs.io/en/latest/build.html
# This takes a while...cross compile if impatient.
$ pip3 install xgboost

# Test...
$ python3
>>> import xgboost
>>> xgboost.__version__
'0.90'
>>>
```

8. Install dlib.
```bash
$ cd /media/mendel

# Update repo.
$ sudo apt-get update

# Install dependencies if needed. 
$ sudo apt-get install build-essential cmake

# Clone dlib repo.
$ git clone https://github.com/davisking/dlib.git

# Build and install. 
$ cd dlib
# -O3 enables auto vectorization optimizations
# so that the compiler automatically uses NEON instructions.
$ python3 setup.py install --set DLIB_NO_GUI_SUPPORT=YES \
--set DLIB_USE_CUDA=NO --compiler-flags "-O3"

# Test...
python3
>>> import dlib
>>> dlib.__version__
'19.17.99'
>>>
```

9. Install face_recognition.
```bash
$ pip3 install face_recognition

# Test...
$ python3
>>> import face_recognition
>>> face_recognition.__version__
'1.2.3'
>>>
```

10. Disable and remove swap.
```bash
$ cd /media/mendel
$ sudo swapoff /swapfile
$ sudo rm -i /swapfile
```

11. Create a directory called *tpu-servers* in ```/media/mendel``` on the Coral dev board.

12. Copy *detect_server_tpu.py* and *config.json* in this directory to ```/media/mendel/tpu-servers```.

13. Create a directory called *models* and another called *labels* in ```/media/mendel/tpu-servers```.

13. Copy the pickled label file (*face_labels.pickle*) to ```/media/mendel/tpu-servers/labels```, and the svm-, and xgd-based face classifiers (*svm_face_recognizer.pickle*, and *xgb_face_recognizer.pickle*, respectively) created in the [face-det-rec train step](../face-det-rec/README.md) to ```/media/mendel/tpu-servers/models```. Alternatively, you can generate facial embeddings and train face classifiers on them directly on the Coral dev board but the xgb-based classifier training takes a long time, see [edge-tpu-servers](https://github.com/goruck/edge-tpu-servers) on how to do this. 

14. Download the TPU face recognition dnn model *MobileNet SSD v2 (Faces)* from [Google Coral](https://coral.withgoogle.com/models/) to ```/media/mendel/tpu-servers/models```.

15. Download both the *MobileNet SSD v2 (COCO)* tpu object detection dnn model and label file from [Google Coral](https://coral.withgoogle.com/models/) to ```/media/mendel/tpu-servers/models``` and ```/media/mendel/tpu-servers/labels``` respectively.

*NB: You can instead use transfer learning to train your own models and use them instead of the Google stock models in the steps above, see [TensorFlow Models with Edge TPU Training](https://github.com/goruck/models/tree/edge-tpu).*

16. Copy the person classification tflite models that have been compiled for the edge TPU from [person-class/train-results](../person-class/train-results) to ```/media/mendel/tpu-servers/models```. See the [person-class README](../person-class/README.md) on how to generate these models. 

17. Mount ZoneMinder's alarm image store on the Dev Board so the server can find the alarm images and process them. The store should be auto-mounted using ```sshfs``` at startup which is done by an entry in ```/etc/fstab```.
```bash
# Setup sshfs.
$ sudo apt-get install sshfs

# Create mount point.
$ sudo mkdir /mnt/nvr

# Setup SSH keys to enable auto login.
# See https://www.cyberciti.biz/faq/how-to-set-up-ssh-keys-on-linux-unix/
$ mkdir -p $HOME/.ssh
$ chmod 0700 $HOME/.ssh
# Create the key pair
$ ssh-keygen -t rsa
# Install the public key on the server hosting ZoneMinder.
$ ssh-copy-id -i $HOME/.ssh/id_rsa.pub lindo@192.168.1.4

# Edit /etc/fstab so that the store is automounted. Here's mine.
$ more /etc/fstab
...
lindo@192.168.1.4:/nvr /mnt/nvr fuse.sshfs auto,user,_netdev,reconnect,uid=1000,gid=1000,IdentityFile=/home/mendel/.ssh/id_rsa,idmap=user,allow_other 0 0

# Test mount the zm store. This should happen at boot from now on. 
$ sudo mount -a
$ ls /mnt/nvr
camera-share  lost+found  zoneminder
```
Note that I have seen cases where fstab fails to mount a network drive at boot when only WLAN is used (LAN is robust). As a failsafe you can force a mount when ever a network interface is started by adding a script to /etc/network/if-up.d as shown below.

```bash
mendel@tpu:/etc/network/if-up.d$ ls
avahi-daemon  ethtool  mount_command  openssh-server  upstart  wpasupplicant
mendel@tpu:/etc/network/if-up.d$ more mount_command 
#!/bin/bash
# Force mount of network drives when an interface comes up.
# Failsafe in case fstab fails to mount a drive becase of network issues.
mount -a
```

18. Edit the [config.json](./config.json) to suit your installation. The configuration parameters are documented in the [detect_servers_tpu.py](./detect_servers_tpu.py) code. Since the TPU detection servers and ZoneMinder are running on different machines make sure both are using the same TCP socket. The parameter *recognizeMode* must be set to either *face* to use the face recognizer or *person* to use the person classifier. 

19. Use systemd to run the server as a Linux service. Edit [detect-tpu.service](./detect-tpu.service) to suit your configuration and copy the file to ```/lib/systemd/system/detect-tpu.service``` on the Coral dev board. Then enable and start the service:
```bash
$ sudo systemctl enable detect-tpu.service && sudo systemctl start detect-tpu.service
```

20. Test the entire setup by editing [detect_servers_test.py](detect_servers_test.py) with paths to test images and running that program.

# Notes
1. Use [evaluate_model.py](./evaluate_model.py) to determine the classification accuracy of the tflite quantized person classifier running on the TPU. 