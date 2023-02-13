**This is a fork of DarkneTZ repo with code modifications to enable non-successive layer protection (Static GradSec)**

This is an application that runs two chunks of non-successive layers of a Deep Neural Network (DNN) model in TrustZone.

This application is based on [DarkneTZ](https://github.com/mofanv/darknetz) and needs to be run with [OP-TEE](https://www.op-tee.org/), an open source framework for Arm TrustZone.

---------------------------
If you find this project useful, please cite this paper:
<code>
@inproceedings{10.1145/3528535.3565255,
author = {Messaoud, Aghiles Ait and Mokhtar, Sonia Ben and Nitu, Vlad and Schiavoni, Valerio},
title = {Shielding Federated Learning Systems against Inference Attacks with ARM TrustZone},
year = {2022},
isbn = {9781450393409},
publisher = {Association for Computing Machinery},
address = {New York, NY, USA},
url = {https://doi.org/10.1145/3528535.3565255},
doi = {10.1145/3528535.3565255},
booktitle = {Proceedings of the 23rd ACM/IFIP International Middleware Conference},
pages = {335–348},
numpages = {14},
keywords = {privacy, federated learning, TrustZone, trusted execution environment},
location = {Quebec, QC, Canada},
series = {Middleware '22}
}
</code>

# Prerequisites
You can run this application with real TrustZone or a simulated one by using QEMU.

**Required System**: Ubuntu-based distributions

**For simulation**, no additional hardware is needed.

**For real TrustZone, an additional board is required**. Raspberry Pi 3, HiKey Board, ARM Juno board, etc. Check this [List](https://optee.readthedocs.io/en/latest/building/devices/index.html#device-specific) for more info.

# Setup
## (1) Set up OP-TEE
1) Follow **step1** ~ **step5** in "**Get and build the solution**" to build the OP-TEE solution.
https://optee.readthedocs.io/en/latest/building/gits/build.html#get-and-build-the-solution

2) **For real boards**: If you are using boards, keep follow **step6** ~ **step7** in the above link to flash the devices. This step is device-specific.

   **For simulation**: If you have chosen QEMU-v7/v8, run the below command to start QEMU console.
```
make run
(qemu)c
```

3) Follow **step8** ~ **step9** to test whether OP-TEE works or not. Run:
```
tee-supplicant -d
xtest
```

Note: you may face OP-TEE related problem/errors during setup, please also free feel to raise issues in [their pages](https://github.com/OP-TEE/optee_os).

## (2) Build Static GradSec
1) clone codes and datasets
```
git clone https://github.com/aghiles-ait/Static_GradSec.git
**git checkout master**
git clone https://github.com/mofanv/tz_datasets.git
```
Let `$PATH_OPTEE$` be the path of OPTEE, `$PATH_gradsec$` be the path of gradsec, and `$PATH_tz_datasets$` be the path of tz_datasets.

2) copy GradSec to example dir
```
mkdir $PATH_OPTEE$/optee_examples/gradsec
cp -a $PATH_gradsec$/. $PATH_OPTEE$/optee_examples/gradsec/
```

3) copy datasets to root dir
```
cp -a $PATH_tz_datasets$/. $PATH_OPTEE$/out-br/target/root/
```

4) rebuild the OP-TEE

**For simulation**, to run `make run` again.

**For real boards**, to run `make flash` to flash the OP-TEE with `darknetz` to your device.

5) after you boot your devices or QEMU, test by the command 
```
darknetp
```
Note: It is NOT `darknetz` here for the command.

You should get the output:
 ```
# usage: ./darknetp <function>
 ```
Awesome! You are ready to run DNN layers in TrustZone.

# Train Models

1) To train a model from scratch 
```
darknetp classifier train -pp_start1 2 -pp_end1 4 -pp_start2 8 -pp_end2 10 cfg/mnist.dataset cfg/mnist_lenet.cfg
```
You can choose the two chunks of protected layers in the TEE by adjusting the argument `-pp_start1`, `-pp_end1`, `-pp_start2` and `-pp_end2`. The order pp_start1 < pp_end1 < pp_start2 < pp_end2 should be respected. The two chunks of protected layers are respectively between pp_start1 and pp_end1 and between pp_start2 and pp_end2.

Note: you may suffer from insufficient secure memory problems when you run this command. The preconfigured secure memory of darknetz is `TA_STACK_SIZE = 1*1024*1024` and `TA_DATA_SIZE = 10*1024*1024` in `ta/user_ta_header_defines.h` file. However, the maximum secure memory size can differ from different devices (typical size is 16 MiB) and maybe not enough. When this happens, you may want to either, configure the needed secure memory to be smaller, or increase the secure memory of the device (for QEMU go to the [link here](https://github.com/OP-TEE/optee_os/issues/2079)).

When everything is ready, you will see output from the Normal World like this:
```
# Prepare session with the TA
# Begin darknet
# mnist_lenet
# 1
layer     filters    size              input                output
    0 conv      6  5 x 5 / 1    28 x  28 x   3   ->    28 x  28 x   6  0.001 BFLOPs
    1 max          2 x 2 / 2    28 x  28 x   6   ->    14 x  14 x   6
    2 conv_TA   6  5 x 5 / 1    14 x  14 x   6   ->    14 x  14 x   6  0.000 BFLOPs
    3 max_TA       2 x 2 / 2    14 x  14 x   6   ->     7 x   7 x   6
    4 connected_TA                          294  ->   120
    5 dropout      p = 0.80                 120  ->   120
    6 connected                             120  ->    84
    7 dropout      p = 0.80                  84  ->    84
    8 connected_TA                           84  ->    10
    9 softmax_TA                                       10
   10 cost_TA                                          10
# Learning Rate: 0.01, Momentum: 0.9, Decay: 5e-05
# 1000
# 28 28
# Loaded: 0.197170 seconds
# 1, 0.050: 0.000000, 0.000000 avg, 0.009999 rate, 3.669898 seconds, 50 images
# Loaded: 0.000447 seconds
# 2, 0.100: 0.000000, 0.000000 avg, 0.009998 rate, 3.651714 seconds, 100 images
...
```

Layers with `_TA` are running in the TrustZone. When the last layer is inside the TEE, you will not see the loss from Normal World. The training loss is calculated based on outputs of the model which belong to the last layer in the TrustZone, so it can only be seen from the Secure World. That is, the output from the Secure World is like this:
```
# I/TA:  loss = 1.62141, avg loss = 1.62540 from the TA
# I/TA:  loss = 1.58659, avg loss = 1.61783 from the TA
# I/TA:  loss = 1.57328, avg loss = 1.59886 from the TA
# I/TA:  loss = 1.52641, avg loss = 1.57889 from the TA
...
```




