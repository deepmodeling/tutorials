# Transfer-learning

## What is transfer-learning

[Transfer Learning](https://builtin.com/data-science/transfer-learning) is the reuse of a pre-trained model on a new problem. It's currently very popular in [deep learning](https://builtin.com/artificial-intelligence/deep-learning) because it can train [deep neural networks](https://builtin.com/data-science/recurrent-neural-networks-and-lstm) with comparatively little data. In our case, transfer learning means re-training a pre-trained DP model to another one by a small dataset, where the size of the new dataset might be orders of magnitude smaller in comparison to that of the original dataset used to train the original model. 

## When to use transfer-learning

Transfer learning has several benefits, where the main advantages are saving training time, better performance of neural networks (in most cases), and not needing a lot of data. The central idea under transfer-learning is that the knowledge of a model learned from a large dataset can be reused.  In DP models, the typical knowledge that can be reused is the embedding net, which encode local atomic configurations into descriptors. For example, the following are some possible scenarios where transfer-learning may be useful:

- Initially, we may generate DFT dataset without van der Waals correction. However, after sometime, we may find that van der Waals correction is important and should be considered.
- Initially, we may generate DFT dataset PBE functional. However, after sometime, we may find that the PBE functional is not accurate enough. Alternatively, we may find that the original kspacing or energy cutoff cannot meet the convergence criterion for some highly distorted configurations.
- We have a big DFT dataset of a material and DP models trained on the dataset in hand, and need a model of another material that is similar to the material. For example, we have a dataset of HfO2 in the [DP library](https://dplibrary.deepmd.net/), and need a model of ZrO2.
- We have a big DFT dataset of a material and DP models trained on the dataset in hand, and need to train a model on other properties by using the [DeePMD-kit](https://github.com/deepmodeling/deepmd-kit) software. Here, the property to be trained is assumed also depending on local atomic configurations.

## How to implement transfer-learning

This tutorial will introduce how to implement potential energy surface (PES) transfer-learning by using the [DP-GEN](https://github.com/deepmodeling/dpgen) software. In [DP-GEN](https://github.com/deepmodeling/dpgen) (version > 0.8.0), the "simplify" module is designed for this purpose. Suppose that we have completed a typical [DP-GEN](https://github.com/deepmodeling/dpgen) flow, and obtained the DFT dataset and four DP models. The workflow of "simplify" is similar to a typical [DP-GEN](https://github.com/deepmodeling/dpgen) process: iteratively training the DP models with the (re-) labeled data (00.train), picking data according to prediction deviations between different models (01.model_devi), and (re-) labeling the picked data (02.fp). Repeat the iterations until convergence is achieved. Then, the relabeled new dataset that is sufficient to refine the DP model is successfully collected.

In the "simplify" mode, the first iteration can be viewed as the initialization process in the conventional [DP-GEN](https://github.com/deepmodeling/dpgen) process, where the 00. train and 01. model_devi are skipped, and some data are randomly picked in 02.fp to be relabeled. The goal of relabeling may be using a different functional, using a different pseudopotential, using different parameters to achieve higher precision, *etc*. From the second iteration on:

- In the training step (00. train), the DP models are modified based on the relabeled data with "init-model" mode by freezing some parameters of the DP models. For example, the parameters of the whole embedding net and the hidden layers of the fitting net can be fixed, and only the parameters of the output layer of the fitting net are trainable. The trainable parameters can be set according to your demand.
- In the exploration step (01. model_devi), the deviations between predictions by the modified models on the original dataset are evaluated. Some of the data points (*e.g.* at most 100) with model deviation exceeding a criterion are randomly selected for relabeling.
- In the labeling step (02.fp), the selected data points are relabeled, and fed to the new dataset. 

The iterations will stop unit no data is picked up. 

## Example: Ag-Au

In this example, we generate the Ag-Au alloy DP model based on DFT-PBE without van der Waals correction. In fact, the van der Waals dispersion correction is necessary to correctly predict lattice parameter and surface formation energy in Ag-Au alloy system. Besides, DFT-PBE-D3 gives qualitatively reasonable results for the surface adsorption behaviors, which is widely used in investigating Ag segregation at the Ag-Au surface, and adatom adsorption/diffusion on Ag or Au surfaces. Therefore, a DP model honest to DFT-PBE-D3 is required for predicting surface behavior at larger scales.

The original DFT-PBE dataset is collected by using command "dpgen collect", making the dataset convenient to read by simplify parameter files. Here, the DFT-PBE dataset is generated from the previous DPGEN iteration. Then, start the process by using command 

```Plain%20Text
dpgen simplify simplify.json machine.json
```

where the parameter file "simplify.json" is similar to that of a typical [DP-GEN](https://github.com/deepmodeling/dpgen) process. The "simplify.json" file is shown and explained in the following. The process converged in only four iterations, where merely 144, 155 and 275 structures were selected to be relabeled for pure Ag, pure Au, and Ag-Au alloys, respectively. The transferred DP-PBE-D3 model agrees with DFT-PBE-D3.

By the way, it is really cheap to do the D3 correction, which only requires the atom coordinates and the box size as the input information. Some codes could do this:

https://chemie.uni-bonn.de/pctc/mulliken-center/software/dft-d3/get-the-current-version-of-dft-d3

https://github.com/MMunibas/PhysNet/blob/master/neural_network/grimme_d3/grimme_d3.py

https://github.com/loriab/dftd3

https://github.com/dftbplus/dftd3-lib

According to the scripts on the last website, we relabeled all the data with DFT-PBE-D3. Therefore, this example can also be done without transfer-learning. However, in other scenarios listed in the part of "when to use transfer learning", transfer-learning is beneficial.

 

Examplary "simplify.json" of Ag-Au system: 

```
{
    "type_map":    [ "Ag", "Au" ],
    "mass_map":    [ 108, 196.967 ],
    "init_data_prefix": "",
    "init_data_sys":  [],
    "pick_data":  "/Ag-Au/simp/collect",
# Use 'dp collect' to put all the data in one fold. 
    "sys_batch_size":  "auto",
    "training_iter0_model_path":  "/Ag-Au/final/00.train/00[0-3]",
# Locate the original DP model.
    "training_init_model":    true,
# Transfer learning is doing 'init-model' based on the original one. 
    "numb_models":  4,
    "train_param":  "input.json",
    "default_training_param" : {
        "_comment": " model parameters",
    "model": {
            "type_map":     ["Ag", "Au"],
            "descriptor" :{
        "type":             "se_a",
        "sel":              [150,150],
        "rcut_smth":        2.00,
        "rcut":             6.00,
        "neuron":           [25, 50, 100],
        "trainable":      false,
# Parameters of embedding net are not trained.
    "resnet_dt":        false,
    "axis_neuron":      12,
    "seed":             0,
                "_comment":         " that's all"
            },
            "fitting_net" : {
    "neuron":           [240, 240, 240],
    "resnet_dt":        true,
    "trainable":      [false, false, false, true],
# The parameters of the output layer of the fitting net are trainable. 
    "seed":             1,
    "_comment":         " that's all"
            },
            "_comment":     " that's all"
  },
  "learning_rate" :{
            "type":         "exp",
            "start_lr":     0.001,
            "stop_lr":      3.51e-8,
            "decay_steps":  2000,
            "_comment":     "that's all"
  },
  "loss" :{
            "start_pref_e": 100,
            "limit_pref_e": 100,
            "start_pref_f": 1,
            "limit_pref_f": 1,
            "start_pref_v": 0.9,
            "limit_pref_v": 1.0,
            "_comment":     " that's all"
  },
  "_comment": " traing controls",
  "training" : {
            "systems":      [],
            "set_prefix":   "set",
            "stop_batch":   400000,
            "batch_size":   2,
            "seed":         1,
            "_comment": " display and restart",
            "_comment": " frequencies counted in batch",
            "disp_file":    "lcurve.out",
            "disp_freq":    2000,
            "numb_test":    2,
            "save_freq":    2000,
            "save_ckpt":    "model.ckpt",
            "load_ckpt":    "model.ckpt",
            "disp_training":true,
            "time_training":true,
            "profiling":    false,
            "profiling_file":"timeline.json",
            "_comment":     "that's all"
},
  "_comment":         "that's all"
},
"fp_style": "vasp",
"fp_skip_bad_box":  "length_ratio:5;height_ratio:5",
"shuffle_poscar":  false,
"fp_task_max":  100,
# At most 100 data points are randomly picked and sent to the labeling step.
"fp_task_min" :  0,
# All the data picked in the first iteration will be sent to relabel. 
    "fp_pp_path":  "vasp_input",
    "fp_pp_files":  ["POTCAR_Ag", "POTCAR_Au"],
    "fp_incar":         "vasp_input/INCAR",

    "use_clusters": false,
    "labeled": false,
    "init_pick_number":100,
    "iter_pick_number":100,
    "e_trust_lo":1e10,
    "e_trust_hi":1e10,
    "f_trust_lo":0.20,
    "f_trust_hi":100.00,
# Once the mode_deviation of the data is higher than f_trust_lo, it is selected as a candidate. Since all the data is selected from the original data set, which is proper for constructing a good DP model,  we do not especially need to exclude the data with bad box or atom overlap.    
    "_comment": " that's all "
}
```

## References

YiNan Wang et al 2022 Modelling Simul. Mater. Sci. Eng. 30 025003

Grimme S, Antony J, Ehrlich S and Krieg H 2010 J. Chem. Phys. 132 154104
