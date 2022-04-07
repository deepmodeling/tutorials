# Hands-on tutorial for DP-GEN (v0.10.3)

Writer: Wenshuo Liang

Verifier: Yibo Wang

## General Introduction
Deep Potential GENerator (DP-GEN) is a package that implements a concurrent learning scheme to generate reliable DP models. Typically, the DP-GEN workflow contains three processes: init, run, and autotest. 

- init: generate the initial training dataset by first-principle calculations.
- run: the main process of DP-GEN, in which the training dataset is enriched and the quality of the DP models is improved automatically.
- autotest: calculate a simple set of properties and/or perform tests for comparison with DFT and/or empirical interatomic potentials.

This is a practical tutorial that aims to help you quickly get command of dpgen run interface.

## Input files
In this tutorial, we take a gas-phase methane molecule as an example. We have prepared input files in dpgen_example/run

Now download the dpgen_example and uncompress it:

```sh
wget https://dp-public.oss-cn-beijing.aliyuncs.com/community/dpgen_example.tar.xz
tar xvf dpgen_example.tar.xz
```
Now go into the dpgen_example/run.
```sh
$ cd dpgen_example/run
$ ls
INCAR_methane  machine.json  param.json  POTCAR_C  POTCAR_H
```

- param.json is the settings for DP-GEN for the current task. It will expalin later.
- machine.json is a task dispatcher where the machine environment and resource requirements are set.[We have expalined here](www.baidu.com)
- INCAR* and POTCAR* are the input file for the VASP package.  All first-principle calculations share the same parameters as the one you set in param.json.

## Run process

We can run DP-GEN easily by:
```sh
$ dpgen run param.json machine.json
```

The run process contains a series of successive iterations. Each iteration is composed of three steps: 

* `exploration`
* `labeling`
* `training`

Accordingly, there are three sub-folders in each iteration: 

* `00.train`
* `01.model_devi`
* `02.fp` 


### param.json

 We provide an example of param.json.

```json
{
     "type_map": ["H","C"],
     "mass_map": [1,12],
     "init_data_prefix": "../",
     "init_data_sys": ["init/CH4.POSCAR.01x01x01/02.md/sys-0004-0001/deepmd"],
     "sys_configs_prefix": "../",
     "sys_configs": [
         ["init/CH4.POSCAR.01x01x01/01.scale_pert/sys-0004-0001/scale/00000/POSCAR"],
         ["init/CH4.POSCAR.01x01x01/01.scale_pert/sys-0004-0001/scale/00001/POSCAR"]
     ],
     "_comment": " that's all ",
     "numb_models": 4,
     "default_training_param": {
         "model": {
             "type_map": ["H","C"],
             "descriptor": {
                 "type": "se_a",
                 "sel": [16,4],
                 "rcut_smth": 0.5,
                 "rcut": 5.0,
                 "neuron": [120,120,120],
                 "resnet_dt": true,
                 "axis_neuron": 12,
                 "seed": 1
             },
             "fitting_net": {
                 "neuron": [25,50,100],
                 "resnet_dt": false,
                 "seed": 1
             }
         },
         "learning_rate": {
             "type": "exp",
             "start_lr": 0.001,
             "decay_steps": 5000
         },
         "loss": {
             "start_pref_e": 0.02,
             "limit_pref_e": 2,
             "start_pref_f": 1000,
             "limit_pref_f": 1,
             "start_pref_v": 0.0,
             "limit_pref_v": 0.0
         },
         "training": {
             "stop_batch": 400000,
             "disp_file": "lcurve.out",
             "disp_freq": 1000,
             "numb_test": 4,
             "save_freq": 1000,
             "save_ckpt": "model.ckpt",
             "disp_training": true,
             "time_training": true,
             "profiling": false,
             "profiling_file": "timeline.json",
             "_comment": "that's all"
         }
     },
     "model_devi_dt": 0.002,
     "model_devi_skip": 0,
     "model_devi_f_trust_lo": 0.05,
     "model_devi_f_trust_hi": 0.15,
     "model_devi_e_trust_lo": 10000000000.0,
     "model_devi_e_trust_hi": 10000000000.0,
     "model_devi_clean_traj": true,
     "model_devi_jobs": [
         {"sys_idx": [0],"temps": [100],"press": [1.0],"trj_freq": 10,"nsteps": 300,"ensemble": "nvt","_idx": "00"},
         {"sys_idx": [1],"temps": [100],"press": [1.0],"trj_freq": 10,"nsteps": 3000,"ensemble": "nvt","_idx": "01"}
     ],
     "fp_style": "vasp",
     "shuffle_poscar": false,
     "fp_task_max": 20,
     "fp_task_min": 5,
     "fp_pp_path": "./",
     "fp_pp_files": ["POTCAR_H","POTCAR_C"],
     "fp_incar": "./INCAR_methane"
}
```

The following is a detailed description of the keywords.

#### Basics keywords (Line 2-3):

| Key         | Type            | Description             |
|-------------|-----------------|-------------------------|
| "type_map"  | List of string  | Atom types              |
| "mass_map"  | List of float   | Standard atom weights.  |


#### Data-related keywords (Line 4-10):
| Key                   | Type            | Description                                                                                                 |
|-----------------------|-----------------|-------------------------------------------------------------------------------------------------------------|
| "init_data_prefix"    | String          | Prefix of initial data directories                                                                          |
| "init_data_sys"       | List of string  | Directories of initial data. You may use either absolute or relative path here.                             |
| "sys_configs_prefix"  | String          | Prefix of sys_configs                                                                                       |
| "sys_configs"         | List of string  | Containing directories of structures to be explored in iterations. Wildcard characters are supported here.  |

#### Training-related keywords (Line 12-58):
| Key                       | Type     | Description                                  |
|---------------------------|----------|----------------------------------------------|
| "numb_models"             | Integer  | Number of models to be trained in 00.train.  |
| "default_training_param"  | Dict     | Training parameters for deepmd-kit.          |

#### Exploration-related keywords (Line 59-69):
| Key                      | Type                    | Description                                                                                                                                                                                                                                   |
|--------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| "model_devi_dt"          | Float                   | Timestep for MD                                                                                                                                                                                                                               |
| "model_devi_skip"        | Integer                 | Number of structures skipped for fp in each MD                                                                                                                                                                                                |
| "model_devi_f_trust_lo"  | Float                   | Lower bound of forces for the selection. If List, should be set for each index in sys_configs, respectively.                                                                                                                                  |
| "model_devi_f_trust_hi"  | Integer                 | Upper bound of forces for the selection. If List, should be set for each index in sys_configs, respectively.                                                                                                                                  |
| "model_devi_f_trust_hi"  | Float or List of float  | Lower bound of virial for the selection. If List, should be set for each index in sys_configs, respectively. Should be used with DeePMD-kit v2.x.                                                                                             |
| "model_devi_f_trust_hi"  | Float or List of float  | Upper bound of virial for the selection. If List, should be set for each index in sys_configs, respectively. Should be used with DeePMD-kit v2.x.                                                                                             |
| "model_devi_clean_traj"  | Boolean or Int          | If type of model_devi_clean_traj is boolean type then it denote whether to clean traj folders in MD since they are too large. If it is Int type, then the most recent n iterations of traj folders will be retained, others will be removed.  |
| "model_devi_jobs"        | List of Dict            | Settings for exploration in 01.model_devi. Each dict in the list corresponds to one iteration. The index of model_devi_jobs exactly accord with index of iterations                                                                           |

#### Labeling-related parameters (Line 70-76):
| Key               | Type            | Description                                                                                                              |
|-------------------|-----------------|--------------------------------------------------------------------------------------------------------------------------|
| "fp_style"        | String          | Software for First Principles. Options include “vasp”, “pwscf”, “siesta” and “gaussian” up to now.                       |
| "shuffle_poscar"  | Boolean         |                                                                                                                          |
| "fp_task_max"     | Integer         | Maximum of structures to be calculated in 02.fp of each iteration.                                                       |
| "fp_task_min"     | Integer         | Minimum of structures to calculate in 02.fp of each iteration.                                                           |
| "fp_pp_path"      | String          | Directory of psuedo-potential file to be used for 02.fp exists.                                                          |
| "fp_pp_files"     | List of string  | Psuedo-potential file to be used for 02.fp. Note that the order of elements should correspond to the order in type_map.  |
| "fp_incar"        | String          | Input file for VASP. INCAR must specify KSPACING and KGAMMA.                                                             |

## Output files

In dpgen_example/run, we can find that a folder and two files are generated automatically. 
```sh
$ ls 
dpgen.log  INCAR_methane  iter.000000  machine.json  param.json  record.dpgen 
```

- `iter.000000` contains the main results that DP-GEN generates in the first iteration.
- `record.dpgen` records the current stage of the run process. 
- `dpgen.log` includes time and iteration information. 
 When the first iteration is completed, the folder structure of `iter.000000` is like this:

```sh
$ tree iter.000000/ -L 1
./iter.000000/
├── 00.train
├── 01.model_devi
└── 02.fp
```
### 00.train
First, we check the folder `iter.000000`/ `00.train`.
```sh
$ tree iter.000000/00.train -L 1
./iter.000000/00.train/
├── 000
├── 001
├── 002
├── 003
├── data.init -> /root/dpgen_example
├── data.iters
├── graph.000.pb -> 000/frozen_model.pb
├── graph.001.pb -> 001/frozen_model.pb
├── graph.002.pb -> 002/frozen_model.pb
└── graph.003.pb -> 003/frozen_model.pb
```

- Folder 00x contains the input and output files of the DeePMD-kit, in which a model is trained.
- graph.00x.pb , linked to 00x/frozen.pb, is the model DeePMD-kit generates. The only difference between these models is the random seed for neural network initialization. 
We may randomly select one of them, like 000.
```sh
$ tree iter.000000/00.train/000 -L 1
./iter.000000/00.train/000
├── checkpoint
├── frozen_model.pb
├── input.json
├── lcurve.out
├── model.ckpt-400000.data-00000-of-00001
├── model.ckpt-400000.index
├── model.ckpt-400000.meta
├── model.ckpt.data-00000-of-00001
├── model.ckpt.index
├── model.ckpt.meta
└── train.log
```

- `input.json` is the settings for deepmd-kit for current task.
- `checkpoint`  is used for restart traning.
- `model.ckpt*` are model related files.
- `frozen_model.pb` is the frozen model. 
- `lcurve.out` records the training accuracy of energies and forces.
- `train.log` includes version, data, hardware information, time, etc.

### 01.model_devi
Then, we check the folder iter.000000/ 01.model_devi.
```sh
$ tree iter.000000/01.model_devi -L 1
./iter.000000/01.model_devi/
├── confs
├── graph.000.pb -> /root/dpgen_example/run/iter.000000/00.train/graph.000.pb
├── graph.001.pb -> /root/dpgen_example/run/iter.000000/00.train/graph.001.pb
├── graph.002.pb -> /root/dpgen_example/run/iter.000000/00.train/graph.002.pb
├── graph.003.pb -> /root/dpgen_example/run/iter.000000/00.train/graph.003.pb
├── task.000.000000
├── task.000.000001
├── task.000.000002
├── task.000.000003
├── task.000.000004
├── task.000.000005
├── task.000.000006
├── task.000.000007
├── task.000.000008
└── task.000.000009
```

- Folder confs contains the initial configurations for LAMMPS MD converted from POSCAR you set in "sys_configs" of param.json. 

- Folder task.000.00000x contains the input and output files of the LAMMPS. We may randomly select one of them, like task.000.000001. 
```sh
$ tree iter.000000/01.model_devi/task.000.000001
./iter.000000/01.model_devi/task.000.000001
├── conf.lmp -> ../confs/000.0001.lmp
├── input.lammps
├── log.lammps
├── model_devi.log
└── model_devi.out
```

- `conf.lmp`, linked to `000.0001.lmp` in folder confs, serves as the initial configuration of MD. 
- `input.lammps` is the input file for LAMMPS.
- `model_devi.out` records the model deviation of concerned labels, energy and force, in MD. It serves as the criterion for selecting which structures and doing first-principle calculations.

By head `model_devi.out`, you will see:
```
$ head -n 5 ./iter.000000/01.model_devi/task.000.000001/model_devi.out
 #  step max_devi_v     min_devi_v     avg_devi_v     max_devi_f     min_devi_f     avg_devi_f 
 0     1.438427e-04   5.689551e-05   1.083383e-04   8.835352e-04   5.806717e-04   7.098761e-04
10     3.887636e-03   9.377374e-04   2.577191e-03   2.880724e-02   1.329747e-02   1.895448e-02
20     7.723417e-04   2.276932e-04   4.340100e-04   3.151907e-03   2.430687e-03   2.727186e-03
30     4.962806e-03   4.943687e-04   2.925484e-03   5.866077e-02   1.719157e-02   3.011857e-02
```
Now we'll concentrate on `max_devi_f`.
Recall that we've set `"trj_freq"` as 10, so every 10 steps the structures are saved. Whether to select the structure depends on its `"max_devi_f"`. If it falls between `"model_devi_f_trust_lo"` (0.05) and `"model_devi_f_trust_hi"` (0.15), DP-GEN will treat the structure as a candidate. Here, only the 30th structure will be selected, whose `"max_devi_f"` is 5.866077e e-02.

### 02.fp
Finally, we check the folder iter.000000/ 02.fp.
```
$ tree iter.000000/02.fp -L 1
./iter.000000/02.fp
├── data.000
├── task.000.000000
├── task.000.000001
├── task.000.000002
├── task.000.000003
├── task.000.000004
├── task.000.000005
├── task.000.000006
├── task.000.000007
├── task.000.000008
├── task.000.000009
├── task.000.000010
├── task.000.000011
├── candidate.shuffled.000.out
├── POTCAR.000
├── rest_accurate.shuffled.000.out
└── rest_failed.shuffled.000.out
```

- `POTCAR` is the input file for VASP generated according to `"fp_pp_files"` of param.json.
- `candidate.shuffle.000.out` records which structures will be selected from last step 01.model_devi.  There are always far more candidates than the maximum you expect to calculate at one time. In this condition, DP-GEN will randomly choose up to `"fp_task_max"` structures and form the folder task.*.
- `rest_accurate.shuffle.000.out` records the other structures where our model is accurate ("max_devi_f" is less than `"model_devi_f_trust_lo"`, no need to calculate any more), 
- `rest_failed.shuffled.000.out` records the other structures where our model is too inaccurate (lager than `"model_devi_f_trust_hi"`, there may be some error).
- `data.000`: After first-principle calculations, DP-GEN will collect these data and change them into the format DeePMD-kit needs. In the next iteration's `00.train`, these data will be trained together as well as initial data.

By cat candidate.shuffled.000.out \| grep task.000.000001, you will see:

```sh
$ cat ./iter.000000/02.fp/candidate.shuffled.000.out | grep task.000.000001
iter.000000/01.model_devi/task.000.000001 190
iter.000000/01.model_devi/task.000.000001 130
iter.000000/01.model_devi/task.000.000001 120
iter.000000/01.model_devi/task.000.000001 150
iter.000000/01.model_devi/task.000.000001 280
iter.000000/01.model_devi/task.000.000001 110
iter.000000/01.model_devi/task.000.000001 30
iter.000000/01.model_devi/task.000.000001 230
```

The `task.000.000001` 30 is exactly what we have just found in `01.model_devi` satisfying the criterion to be calculated again.
After the first iteration, we check the contents of dpgen.log and record.dpgen.

```sh
$ cat dpgen.log
2022-03-07 22:12:45,447 - INFO : start running
2022-03-07 22:12:45,447 - INFO : =============================iter.000000==============================
2022-03-07 22:12:45,447 - INFO : -------------------------iter.000000 task 00--------------------------
2022-03-07 22:12:45,451 - INFO : -------------------------iter.000000 task 01--------------------------
2022-03-08 00:53:00,179 - INFO : -------------------------iter.000000 task 02--------------------------
2022-03-08 00:53:00,179 - INFO : -------------------------iter.000000 task 03--------------------------
2022-03-08 00:53:00,187 - INFO : -------------------------iter.000000 task 04--------------------------
2022-03-08 00:57:04,113 - INFO : -------------------------iter.000000 task 05--------------------------
2022-03-08 00:57:04,113 - INFO : -------------------------iter.000000 task 06--------------------------
2022-03-08 00:57:04,123 - INFO : system 000 candidate :     12 in    310   3.87 %
2022-03-08 00:57:04,125 - INFO : system 000 failed    :      0 in    310   0.00 %
2022-03-08 00:57:04,125 - INFO : system 000 accurate  :    298 in    310  96.13 %
2022-03-08 00:57:04,126 - INFO : system 000 accurate_ratio:   0.9613    thresholds: 1.0000 and 1.0000   eff. task min and max   -1   20   number of fp tasks:     12
2022-03-08 00:57:04,154 - INFO : -------------------------iter.000000 task 07--------------------------
2022-03-08 01:02:07,925 - INFO : -------------------------iter.000000 task 08--------------------------
2022-03-08 01:02:07,926 - INFO : failed tasks:      0 in     12    0.00 % 
2022-03-08 01:02:07,949 - INFO : failed frame:      0 in     12    0.00 % 
```

It can be found that 310 structures are generated in iter.000000, in which 12 structures are collected for first-principle calculations.
```sh
$ cat record.dpgen
0 0
0 1
0 2
0 3
0 4
0 5
0 6
0 7
0 8
```

Each line contains two numbers: the first is the index of iteration, and the second, ranging from 0 to 9, records which stage in each iteration is currently running.

| Index of iterations  | "Stage in eachiteration "   | Process          |
|----------------------|-----------------------------|------------------|
| 0                    | 0                           | make_train       |
| 0                    | 1                           | run_train        |
| 0                    | 2                           | post_train       |
| 0                    | 3                           | make_model_devi  |
| 0                    | 4                           | run_model_devi   |
| 0                    | 5                           | post_model_devi  |
| 0                    | 6                           | make_fp          |
| 0                    | 7                           | run_fp           |
| 0                    | 8                           | post_fp          |

If the process of DP-GEN stops for some reason, DP-GEN will automatically recover the main process by record.dpgen. You may also change it manually for your purpose, such as removing the last iterations and recovering from one checkpoint.
After all iterations, we check the structure of dpgen_example/run 
```sh
$ tree ./ -L 2
./
├── dpgen.log
├── INCAR_methane
├── iter.000000
│   ├── 00.train
│   ├── 01.model_devi
│   └── 02.fp
├── iter.000001
│   ├── 00.train
│   ├── 01.model_devi
│   └── 02.fp
├── iter.000002
│   └── 00.train
├── machine.json
├── param.json
└── record.dpgen
```

and contents of `dpgen.log`.
```sh
$ cat cat dpgen.log | grep system
2022-03-08 00:57:04,123 - INFO : system 000 candidate :     12 in    310   3.87 %
2022-03-08 00:57:04,125 - INFO : system 000 failed    :      0 in    310   0.00 %
2022-03-08 00:57:04,125 - INFO : system 000 accurate  :    298 in    310  96.13 %
2022-03-08 00:57:04,126 - INFO : system 000 accurate_ratio:   0.9613    thresholds: 1.0000 and 1.0000   eff. task min and max   -1   20   number of fp tasks:     12
2022-03-08 03:47:00,718 - INFO : system 001 candidate :      0 in   3010   0.00 %
2022-03-08 03:47:00,718 - INFO : system 001 failed    :      0 in   3010   0.00 %
2022-03-08 03:47:00,719 - INFO : system 001 accurate  :   3010 in   3010 100.00 %
2022-03-08 03:47:00,722 - INFO : system 001 accurate_ratio:   1.0000    thresholds: 1.0000 and 1.0000   eff. task min and max   -1    0   number of fp tasks:      0
```
It can be found that 3010 structures are generated in `iter.000001`, in which no structure is collected for first-principle calculations. Therefore, the final models are not updated in iter.000002/00.train. 


