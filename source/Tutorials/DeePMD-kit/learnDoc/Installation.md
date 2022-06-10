# Installation

There are three easy methods to install DeePMD-kit：
- Install off-line packages
- Install with conda
- Install with docker

Users can choose a suitable method depending on the machine environment. The following is a detailed description of these three methods.

## Install off-line packages

Users can use the offline packages to install the DeePMD-kit if their machine cannot be connected to the internet. Both CPU and GPU version offline packages are available on [the Releases page](https://github.com/deepmodeling/deepmd-kit/releases).

Some GPU version off-line packages are splited into two files due to size limit of GitHub. Users can merge them into one with the `cat` command and then install the DeePMD-kit software with the `bash` command.

```sh
$ cat deepmd-kit-2.0.0-cuda11.3_gpu-Linux-x86_64.sh.0 deepmd-kit-2.0.0-cuda11.3_gpu-Linux-x86_64.sh.1 > deepmd-kit-2.0.0-cuda11.3_gpu-Linux-x86_64.sh
$ bash deepmd-kit-2.0.0-cuda11.3_gpu-Linux-x86_64.sh
```

During the installation, users need to specify the installation path of the DeePMD-kit. It is assumed that the user chose to install DeePMD-kit at "/root/deepmd-kit".  Then, users need to configure the environment variable.

```sh
$ export PATH="/root/deepmd-kit/bin/:$PATH"
```

Users should remember to configure the environment variable after opening a new terminal. Users can also add the above line into the bashrc file, which is not explained here.

## Install with conda

Users can use `conda` to install the DeePMD-kit if their machine can be connected to the internet. Before installing DeePMD-kit, users need to install [Anaconda](https://www.anaconda.com/products/individual#download-section) or [Miniconda](https://docs.conda.io/en/latest/miniconda.html) and activate the conda  environment.

Both the CPU and GPU versions of DeePMD-kit can be installed via conda. Users can create an environment that contains the CPU version of DeePMD-kit and LAMMPS.

```sh
(base)$ conda create -n deepmd deepmd-kit=*=*cpu libdeepmd=*=*cpu lammps-dp -c https://conda.deepmodeling.org
```

or create an environment that contains the GPU version of DeePMD-kit and LAMMPS.

```sh
(base)$ conda create -n deepmd deepmd-kit=*=*gpu libdeepmd=*=*gpu lammps-dp cudatoolkit=11.3 horovod -c https://conda.deepmodeling.org
```

The environment also contains the [CUDA Toolkit](https://docs.nvidia.com/deploy/cuda-compatibility/index.html#binary-compatibility__table-toolkit-driver). Users could change the CUDA Toolkit version from 10.1 or 11.3.
The latest version of DeePMD-kit will be installed by the above command. Users may want to specify the DeePMD-kit version such as 2.0.0 using

```sh
(base)$ conda create -n deepmd deepmd-kit=2.0.0=*cpu libdeepmd=2.0.0=*cpu lammps-dp=2.0.0 horovod -c https://conda.deepmodeling.org
```

Before using the DeePMD-kit, users need to ensure that the environment is active. Users can enable the deepmd environment using

```sh
(base)$ conda activate deepmd
```

When the environment is activated, (base) will be converted to (deepmd) on the left in the terminal. For example,

```sh
(deepmd)$
```

## Install with docker

Users can also use `docker` to install the DeePMD-kit if their machine can be connected to the internet.

To pull the CPU version:

```sh
$ docker pull ghcr.io/deepmodeling/deepmd-kit:2.0.0_cpu
```

To pull the GPU version:

```sh
$ docker pull ghcr.io/deepmodeling/deepmd-kit:2.0.0_cuda10.1_gpu
```

To pull the ROCm version:

```sh
$ docker pull deepmodeling/dpmdkit-rocm:dp2.0.3-rocm4.5.2-tf2.6-lmp29Sep2021
```

## Verify the installation

If the installation is successful, DeePMD-kit (`dp`) and LAMMPS (`lmp`) will be available to execute.`mpirun` is also available considering users may want to train models or run LAMMPS in parallel. To verify the installation, users can execute

```sh
$ dp -h
```

the terminal will show the help information like

```sh
usage: dp [-h] [--version]{config,transfer,train,freeze,test,compress,doc-traininput,model-devi,convert-from}

DeePMD-kit: A deep learning package for many-body potential energy representation and molecular dynamics
...
```

Note that users can also install the DeePMD-kit software from the source code, but this process is relatively complex. A detailed description is presented in ['Install from source code'](https://docs.deepmodeling.org/projects/deepmd/en/master/install/install-from-source.html) Section of DeePMD-kit’s documentation.
