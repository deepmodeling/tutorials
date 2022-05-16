
#  Install DP-GEN

There are various methods to install DP-GEN. 

## Install with off-line packages
 Offline packages of DP-GEN are available in [the Releases page](https://github.com/deepmodeling/dpgen/releases). Once the off-line package has been downloaded, one can install DP-GEN easily by
```sh
bash dpgen-0.10.2-Linux-x86_64.sh
```
## Install with source code
One can download the source code of dpgen by
```sh
git clone https://github.com/deepmodeling/dpgen.git
```
then you can install DP-GEN easily by:
```sh
cd dpgen
pip install --user .
```
With this command, the dpgen executable is install to  `$HOME/.local/bin/dpgen`. You may want to export the  `PATH`  by
```sh
export PATH=$HOME/.local/bin:$PATH
```
## Install with pip

One can install DP-GEN directly by
```sh
pip install dpgen
```

After your installation, DP-GEN (`dpgen`) will be available to execute.  To test if the installation is successful, you may execute
```sh
dpgen -h
```
