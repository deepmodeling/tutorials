# Hands-on tutorial for DeePMD-kit(v2.0.3)
This tutorial will introduce you to the basic usage of the DeePMD-kit, including data preparation, training/freezing/compressing, testing, and running molecular dynamics simulations with LAMMPS.
Typically the DeePMD-kit workflow contains three parts: data preparation, training/freezing/compressing/testing, and molecular dynamics.
The folder structure of this tutorial is like this:

    $ ls
    00.data 01.train 02.lmp

where the folder 00.data contains the data, the folder 01.train contains an example input script to train a model with DeePMD-kit, and the folder 02.lmp contains LAMMPS example script for molecular dynamics simulation.

## Data preparation
The training data of the DeePMD-kit contains the atom type, the simulation box, the atom coordinate, the atom force, the system energy, and the virial. A snapshot of a molecular system that has this information is called a frame. A system of data includes many frames that share the same number of atoms and atom types. For example, a molecular dynamics trajectory can be converted into a system of data, with each time step corresponding to a frame in the system.

The DeePMD-kit adopts a compressed data format. All training data should first be converted into this format and can then be used by DeePMD-kit. The data format is explained in detail in the DeePMD-kit manual that can be found in the DeePMD-kit official Github site http://www.github.com/deepmodeling/deepmd-kit.

We provide a convenient tool named dpdata for converting the data produced by VASP, Gaussian, Quantum-Espresso, ABACUS, and LAMMPS into the compressed format of DeePMD-kit. It needs to be noted that dpdata only works with python 3.x. 

As an example, go to the data folder:

    $ cd data
    $ ls 
    OUTCAR

The OUTCAR was produced by an ab-initio molecular dynamics (AIMD) simulation of a gas phase methane molecule using VASP. Now start an interactive python environment, for example

    $ python

then execute the following commands:

    import dpdata 
    import numpy as np
    sys = dpdata.LabeledSystem('OUTCAR', fmt = 'vasp/outcar') 
    print('# the system contains %d frames' % sys.get_nframes()) 
    sys.to_deepmd_npy('00.data', set_size = 40, prec = np.float32)

The commands import a system of data from the OUTCAR (with format vasp/outcar ), and then dump it into the compressed format (numpy compressed arrays). The output folder 00.data.

    $ ls 00.data
    set.000 set.001 set.002 set.003 set.004 type.raw type_map.raw

The data system that has 200 frames is split into 5 sets, each of which has 40 frames. The parameter set_size specifies the set size. The parameter prec specifies the precision of the floating point number.

    $ cat 00.data/type.raw 
    H C

Since all frames in the system have the same atom types and atom numbers, we only need to specify the type information once for the whole system

    $ cat 00.data/type_map.raw 
    0 0 0 0 1

where atom H is given type 0, and atom C is given type 1.

## Training
### Prepare input script 
Once the data preparation is done, we can go on with training. Now go to the training directory

    $ cd ../01.train
    $ ls 
    input.json

where input.json gives you an example training script. The options are explained in detail in the DeePMD-kit manual, so they are not comprehensively explained. 

In the model section, the parameters of embedding and fitting networks are specified.

    "model":{
	"type_map":		["H", "C"],					# the name of each type of atom
    	"descriptor":{
    		"type": 		"se_e2_a",  			# full relative coordinates are used
    		"rcut":  		6.00,				# cut-off radius
    		"rcut_smth": 		0.50, 				# where the smoothing starts
    		"sel":			[4, 1], 			# the maximum number of type i atoms in the cut-off radius
    		"neuron":		[10, 20, 40],  			# size of the embedding neural network
    		"resnet_dt":		false,
    		"axis_neuron":		4,				# the size of the submatrix of G (embedding matrix)
			"seed": 		1,
			"_comment":		"that's all"
		},
		"fitting_net":{
			"neuron": 		[100, 100, 100],		# size of the fitting neural network
			"resnet_dt":  		true,
			"seed": 		1,
			"_comment":		"that's all"
		},
		"_comment":	"that's all"'
	},

The se\_e2\_a descriptor is used to train the DP model. The item neurons set the size of the embedding and fitting network to [10, 20, 40] and [100, 100, 100], respectively. The components in $\tilde{\mathcal{R}}^{i}$ to smoothly go to zero from 0.5 to 6 Å.

The following are the parameters that specify the learning rate and loss function.

	"learning_rate" :{
		"type":			"exp",
		"decay_steps":		5000,
		"start_lr":		0.001,    
		"stop_lr":		3.51e-8,
		"_comment":		"that's all"
	},
	"loss" :{
		"type":			"ener",
		"start_pref_e":		0.02,
		"limit_pref_e":		1,
		"start_pref_f":		1000,
		"limit_pref_f":		1,
		"start_pref_v":		0,
		"limit_pref_v":		0,
		"_comment":		" that's all"
    },

In the loss function, pref\_e increases from 0.02 to 1 $\mathrm{eV}^{-2}$, and pref\_f decreases from 1000 to 1 Å$^{2}$ $\mathrm{eV}^{-2}$ progressively, which means that the force term dominates at the beginning, while energy and virial terms become important at the end. This strategy is very effective and reduces the total training time. pref_v is set to 0 $\mathrm{eV}^{-2}$, indicating that no virial data are included in the training process. The starting learning rate, stop learning rate, and decay steps are set to 0.001, 3.51e-8, and 5000, respectively.
The training parameters are given in the following

	"training" : {
		"training_data": {
			"systems":          	["../00.data/data_0/",		# location of the training data
						"../00.data/data_1/", 
						"../00.data/data_2/"],
			"batch_size":		"auto",                       
			"_comment":		"that's all"
		},
		"validation_data":{
			"systems":		["../00.data/data_3"],
			"batch_size":		"auto",				# automatically determined
			"numb_btch":		1,
			"_comment":		"that's all"
		},
		"numb_steps":       		100000,				# Number of training batch             
		"seed":				10,
		"disp_file":			"lcurve.out",
		"disp_freq":			1000,
		"save_freq":			10000,
	},

We reshaped the structure of the data, splitting them into a training data and a validation data. The training data has 3 systems and validation data has 1 system. The model is trained for $10^6$ steps.

### Train a model 
After the training script is prepared, we can start the training with DeePMD-kit by simply running

    $ dp train input.json

On the screen, you see the information of the data system(s)

	DEEPMD INFO      ----------------------------------------------------------------------------------------------------
	DEEPMD INFO      ---Summary of DataSystem: training     -------------------------------------------------------------
	DEEPMD INFO      found 3 system(s):
	DEEPMD INFO                              system        natoms        bch_sz        n_bch          prob        pbc
	DEEPMD INFO                  ../00.data/data_0/             5             7            5         0.250          T
	DEEPMD INFO                  ../00.data/data_1/             5             7           10         0.500          T
	DEEPMD INFO                  ../00.data/data_2/             5             7            5         0.250          T
	DEEPMD INFO      -----------------------------------------------------------------------------------------------------
	DEEPMD INFO      ---Summary of DataSystem: validation   --------------------------------------------------------------
	DEEPMD INFO      found 1 system(s):
	DEEPMD INFO                               system        natoms        bch_sz        n_bch          prob        pbc
	DEEPMD INFO                   ../00.data/data_3/             5             7            5         1.000          T

and the starting and final learning rate of this training

	DEEPMD INFO      start training at lr 1.00e-03 (== 1.00e-03), decay_step 5000, decay_rate 0.950006, final lr will be 3.51e-08

If everything works fine, you will see, on the screen, information printed every 1000 steps, like

	DEEPMD INFO    batch    1000 training time 7.61 s, testing time 0.01 s
	DEEPMD INFO    batch    2000 training time 6.46 s, testing time 0.01 s
	DEEPMD INFO    batch    3000 training time 6.50 s, testing time 0.01 s
	DEEPMD INFO    batch    4000 training time 6.44 s, testing time 0.01 s
	DEEPMD INFO    batch    5000 training time 6.49 s, testing time 0.01 s
	DEEPMD INFO    batch    6000 training time 6.46 s, testing time 0.01 s
	DEEPMD INFO    batch    7000 training time 6.24 s, testing time 0.01 s
	DEEPMD INFO    batch    8000 training time 6.39 s, testing time 0.01 s
	DEEPMD INFO    batch    9000 training time 6.72 s, testing time 0.01 s
	DEEPMD INFO    batch   10000 training time 6.41 s, testing time 0.01 s
	DEEPMD INFO    saved checkpoint model.ckpt

They present the training and testing time counts. At the end of the 10000th batch, the model is saved in Tensorflow's checkpoint file model.ckpt. At the same time, the training and testing errors are presented in file lcurve.out.

	$ head -n 2 lcurve.out
	#step       rmse_val       rmse_trn       rmse_e_val       rmse_e_trn       rmse_f_val       rmse_f_trn           lr
	0           1.34e+01       1.47e+01         7.05e-01         7.05e-01         4.22e-01         4.65e-01     1.00e-03

and

	$ tail -n 2 lcurve.out
	999000      1.24e-01       1.12e-01         5.93e-04         8.15e-04         1.22e-01         1.10e-01      3.7e-08
	1000000     1.31e-01       1.04e-01         3.52e-04         7.74e-04         1.29e-01         1.02e-01      3.5e-08

Volumes 4, 5 and 6, 7 present energy and force training and testing errors, respectively. It is demonstrated that after 140,000 steps of training, the energy testing error is less than 1 meV and the force testing error is around 120 meV/Å. It is also observed that the force testing error is systematically (but slightly) larger than the training error, which implies a slight over-fitting to the rather small dataset.

When the training process is stopped abnormally, we can restart the training from the provided checkpoint by simply running

	$ dp train  --restart model.ckpt  input.json

In the lcurve.out, you can see the training and testing errors, like
	
	538000      3.12e-01       2.16e-01         6.84e-04         7.52e-04         1.38e-01         9.52e-02      4.1e-06
	538000      3.12e-01       2.16e-01         6.84e-04         7.52e-04         1.38e-01         9.52e-02      4.1e-06
	539000      3.37e-01       2.61e-01         7.08e-04         3.38e-04         1.49e-01         1.15e-01      4.1e-06
	 #step      rmse_val       rmse_trn       rmse_e_val       rmse_e_trn       rmse_f_val       rmse_f_trn           lr
	530000      2.89e-01       2.15e-01         6.36e-04         5.18e-04         1.25e-01         9.31e-02      4.4e-06
	531000      3.46e-01       3.26e-01         4.62e-04         6.73e-04         1.49e-01         1.41e-01      4.4e-06

Note that input.json needs to be consistent with the previous one.

### Freeze and Compress a model 
At the end of the training, the model parameters saved in TensorFlow's checkpoint file should be frozen as a model file that is usually ended with extension .pb. Simply execute

	$ dp freeze -o graph.pb
	DEEPMD INFO    Restoring parameters from ./model.ckpt-1000000
	DEEPMD INFO    1264 ops in the final graph

and it will output a model file named graph.pb in the current directory. The graph.pb can be compressed in the following way:

	$ dp compress -i graph.pb -o graph-compress.pb
	DEEPMD INFO    stage 1: compress the model
	DEEPMD INFO    built lr
	DEEPMD INFO    built network
	DEEPMD INFO    built training
	DEEPMD INFO    initialize model from scratch
	DEEPMD INFO    finished compressing
	DEEPMD INFO    
	DEEPMD INFO    stage 2: freeze the model
	DEEPMD INFO    Restoring parameters from model-compression/model.ckpt
	DEEPMD INFO    840 ops in the final graph

and it will output a model file named graph-compress.pb.

### Test a model 
We can check the quality of the trained model by running

	$ dp test -m graph-compress.pb -s ../00.data/data_3 -n 40

On the screen you see the information of the prediction errors of validation data.

	DEEPMD INFO    # number of test data	: 40 
	DEEPMD INFO    Energy RMSE		: 3.168050e-03 eV
	DEEPMD INFO    Energy RMSE/Natoms	: 6.336099e-04 eV
	DEEPMD INFO    Force  RMSE		: 1.267645e-01 eV/A
	DEEPMD INFO    Virial RMSE		: 2.494163e-01 eV
	DEEPMD INFO    Virial RMSE/Natoms	: 4.988326e-02 eV
	DEEPMD INFO    # ----------------------------------------------- 

## Run MD with LAMMPS 

Now let's switch to the lammps directory to check the necessary input files for running DeePMD with LAMMPS.

    $ cd ../02.lmp

Firstly, we soft-link the output model in the training directory to the current directory

	$ ln -s ../01.train/graph-compress.pb

Then we have three files

	$ ls
	conf.lmp  graph-compress.pb  in.lammps

where conf.lmp gives the initial configuration of a gas phase methane MD simulation, and the file in.lammps is the lammps input script. One may check in.lammps and finds that it is a rather standard LAMMPS input file for a MD simulation, with only two exception lines:

	pair_style  graph-compress.pb
	pair_coeff  * *

where the pair style deepmd is invoked and the model file graph-compress.pb is provided, which means the atomic interaction will be computed by the DP model that is stored in the file graph-compress.pb.

One may execute lammps in the standard way

	$ lmp  -i  in.lammps

After waiting for a while, the MD simulation finishes, and the log.lammps and ch4.dump files are generated. They store thermodynamic information and the trajectory of the molecule, respectively. One may want to visualize the trajectory by, e.g. ovito

	$ ovito ch4.dump

to check the evolution of the molecular configuration.
