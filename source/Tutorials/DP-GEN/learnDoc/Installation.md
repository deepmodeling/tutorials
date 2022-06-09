# Install DP-GEN

There are various methods to install DP-GEN. Users can choose a suitable method depending on the machine environment. The following is a detailed description of two easy methods.

## Install with pip

Users can use pip to install the DP-GEN if their machine can be connected to the internet. One can install DP-GEN directly by

```sh
pip install dpgen
```

## Install with source code
Users can use the source code to install the DP-GEN if their machine cannot be connected to the internet. One can download the source code of DP-GEN by

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

## Verify the installation

If the installation is successful, DP-GEN (`dpgen`) will be available to execute.  To test if the installation is successful, you may execute

```sh
dpgen -h
```

the terminal will show the help information like
```sh
usage: dpgen [-h]
             {init_surf,init_bulk,auto_gen_param,init_reaction,run,run/report,collect,simplify,autotest,db} 
	     ...

dpgen is a convenient script that uses DeepGenerator to prepare initial data, drive DeepMDkit and analyze results. This script works based on several sub-commands with their own options. To see the options for the sub-commands, type "dpgen sub-command -h".

```
