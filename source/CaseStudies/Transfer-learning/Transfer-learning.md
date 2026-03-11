# Transfer-learning

## What is transfer-learning

[Transfer Learning](https://builtin.com/data-science/transfer-learning) is the reuse of a pre-trained model on a new problem. It is highly popular in [deep learning](https://builtin.com/artificial-intelligence/deep-learning) because it allows you to train complex [deep neural networks](https://builtin.com/data-science/recurrent-neural-networks-and-lstm) using comparatively little data.

In the context of Deep Potential (DP) models, transfer learning means taking a model pre-trained on a massive dataset and fine-tuning it to a new target using a much smaller dataset. Often, the new dataset is orders of magnitude smaller than the original one. This works because the fundamental physics learned by the original model (specifically the embedding network, which encodes local atomic configurations into descriptors) can be reused, leaving only the fitting network to be adapted to the new target.

## When to use transfer-learning

The primary advantages of transfer learning are saving immense amounts of training (and data generation) time, achieving better neural network performance, and drastically reducing the need for expensive high-fidelity data.

Consider transfer learning in the following scenarios:

- Upgrading Physical Accuracy: You generated a massive DFT dataset using standard PBE but later realize van der Waals corrections, higher cutoffs, denser k-spacing, or a more advanced functional (like HSE or SCAN) are necessary for accurate results.

- Expanding to Similar Chemical Spaces: You have a massive DFT dataset and a trained DP model for a specific material, and you need a model for a chemically similar material. For example, adapting a well-trained HfO₂ model from the [AIS Square](https://www.aissquare.com/datasets?page=1&type=datasets) to simulate ZrO₂.

- Training for New Properties: You have a large dataset and DP model for potential energy surfaces (PES) and want to train a model to predict other configuration-dependent properties (e.g., dipole moments or polarizability) using the [DeePMD-kit](https://github.com/deepmodeling/deepmd-kit) software.

## How to implement transfer-learning

To perform transfer learning, you essentially need to complete two distinct steps: obtaining the new target dataset and fine-tuning the pre-trained model.

### Step 1: Obtaining the New Dataset
How you get your new dataset depends on data availability and the computational cost of your target accuracy. We recommend evaluating the following three approaches:

#### Approach A: Utilizing Public Datasets (When data is already available)
If high-fidelity data for your target system already exists, you can skip calculations entirely. You can directly download ready-to-use datasets from community platforms like [AIS Square](https://www.aissquare.com/datasets?page=1&type=datasets) and proceed immediately to fine-tuning your model.

#### Approach B: Full Dataset Relabeling (When calculation is cheap)
If your target data isn't publicly available but the new property or correction is computationally inexpensive to calculate (e.g., empirical D3 corrections), you do not need complex workflows. Simply take your original large dataset, recalculate the energies/forces for every structure using the new method, and perform transfer learning on this fully updated dataset.

#### Approach C: Selective Relabeling via Active Learning (When calculation is expensive)
If public data is unavailable and your new target requires an expensive calculation (like hybrid functionals or Coupled Cluster), relabeling the entire dataset is impossible. In this case, you need to intelligently select a tiny, highly representative subset of your original data to relabel.
This is where the [simplify](https://tutorials.deepmodeling.com/en/latest/Tutorials/DP-GEN/learnDoc/DP-GEN_handson.html#simplify) module in the [DP-GEN](https://github.com/deepmodeling/dpgen) software (version > 0.8.0) comes in. [simplify](https://tutorials.deepmodeling.com/en/latest/Tutorials/DP-GEN/learnDoc/DP-GEN_handson.html#simplify) is an automated active-learning workflow designed specifically to hunt down the most valuable structures for transfer learning. 

### Step 2: Fine-Tuning the Model
Regardless of how you obtained your new dataset (Approach A, B, or C), the actual transfer learning is executed within [DeePMD-kit](https://github.com/deepmodeling/deepmd-kit). By utilizing the [fine-tune](https://docs.deepmodeling.com/projects/deepmd/en/stable/train/finetuning.html) feature in your training parameter file, you load the pre-trained model with a strict configuration: freeze the embedding net and only train the fitting net to adapt to your new high-fidelity data.

## Example: Ag-Au

In this example, we want to upgrade an Ag-Au alloy DP model originally trained on DFT-PBE (without van der Waals correction) to DFT-PBE-D3.
The van der Waals dispersion correction is necessary to correctly predict lattice parameters and surface formation energies in the Ag-Au system. Furthermore, DFT-PBE-D3 gives qualitatively reasonable results for surface adsorption behaviors, which is critical for investigating Ag segregation at the Ag-Au surface. Therefore, a DP model honest to DFT-PBE-D3 is required.

Because we already have the original DFT-PBE dataset (collected via dpgen collect), we have two distinct ways to obtain our new D3 dataset for transfer learning:

### Approach 1: D3 Relabeling

Because adding a D3 empirical correction is computationally trivial—requiring only the atomic coordinates and the box size as input information—we don't necessarily need a complex active learning loop. We can simply relabel the entire dataset using available D3 codes. Some scripts and libraries that can do this rapidly include:

[DFT-D3 (Bonn University)](https://chemie.uni-bonn.de/pctc/mulliken-center/software/dft-d3/get-the-current-version-of-dft-d3)

[PhysNet Grimme D3 implementation](https://github.com/MMunibas/PhysNet/blob/master/neural_network/grimme_d3/grimme_d3.py)

[loriab/dftd3](https://github.com/loriab/dftd3)

[dftbplus/dftd3-lib](https://github.com/dftbplus/dftd3-lib)

Using the scripts from the above website, you could relabel all the data with DFT-PBE-D3 and then fine-tune your model.

### Approach 2: The simplify Workflow

If we want to test the [simplify](https://tutorials.deepmodeling.com/en/latest/Tutorials/DP-GEN/learnDoc/DP-GEN_handson.html#simplify) workflow, or if we pretend for a moment that D3 calculations are highly expensive, we can use DP-GEN to extract a minimal, high-value dataset.

#### Preparation

First, the original DFT-PBE dataset generated from previous DP-GEN iterations needs to be gathered. This is done using the command ```dpgen collect```, which formats the dataset making it convenient for the simplify parameter files to read.
You will also need a parameter file, typically named ```simplify.json```. This file is formatted similarly to a typical DP-GEN process config, but it is specifically designed to guide the active learning loop to hunt for the structures that represent the biggest difference between PBE and PBE-D3.

#### Execution and Workflow

With the data and parameter file ready, you start the process using the command:

```Plain%20Text
dpgen simplify simplify.json machine.json
```

Once initiated, the [simplify](https://tutorials.deepmodeling.com/en/latest/Tutorials/DP-GEN/learnDoc/DP-GEN_handson.html#simplify) active learning loop automatically iterates through these steps:

- Initialization: 00.train and 01.model_devi are skipped. A random, small batch of structures is picked and relabeled (02.fp) using the new, expensive method.

- Training (00.train): DP models are [fine-tuned](https://docs.deepmodeling.com/projects/deepmd/en/stable/train/finetuning.html) on the relabeled data, freezing parameters like the embedding net while keeping the output layers of the fitting net trainable.

- Exploration (01.model_devi): The fine-tuned models predict properties across the entire original dataset. Structures where the models severely disagree (high model deviation) are flagged as areas where the transfer learning needs more data.

- Labeling (02.fp): A small number of these high-deviation structures are sent to be relabeled.

This loop repeats until the models confidently predict the new properties across the entire original dataset space without needing more data.

#### Results

In a real test, this process converged in only four iterations. Astonishingly, merely 144, 155, and 275 structures were selected to be relabeled for pure Ag, pure Au, and Ag-Au alloys, respectively. By fine-tuning the pre-trained PBE model on this tiny handful of newly labeled structures, the transferred DP-PBE-D3 model perfectly agreed with full DFT-PBE-D3 calculations.

Examplary "simplify.json" of Ag-Au system: 

```
{
    "type_map": ["Ag", "Au"],
    "mass_map": [108, 196.967],
    "init_data_prefix": "",
    "init_data_sys": [],
    "pick_data": "/Ag-Au/simp/collect",
    # Use 'dp collect' to put all the data in one fold. 
    "sys_batch_size": "auto",
    "training_iter0_model_path": "/Ag-Au/final/00.train/00[0-3]",
    # Locate the original DP model.
    "training_init_model": true,
    # Transfer learning is doing 'init-model' based on the original one. 
    "numb_models": 4,
    "train_param": "input.json",
    "default_training_param": {
        "_comment": " model parameters",
        "model": {
            "type_map": ["Ag", "Au"],
            "descriptor": {
                "type": "se_a",
                "sel": [150, 150],
                "rcut_smth": 2.00,
                "rcut": 6.00,
                "neuron": [25, 50, 100],
                "trainable": false,
                # Parameters of embedding net are not trained.
                "resnet_dt": false,
                "axis_neuron": 12,
                "seed": 0,
                "_comment": " that's all"
            },
            "fitting_net": {
                "neuron": [240, 240, 240],
                "resnet_dt": true,
                "trainable": [false, false, false, true],
                # The parameters of the output layer of the fitting net are trainable. 
                "seed": 1,
                "_comment": " that's all"
            },
            "_comment": " that's all"
        },
        "learning_rate": {
            "type": "exp",
            "start_lr": 0.001,
            "stop_lr": 3.51e-8,
            "decay_steps": 2000,
            "_comment": "that's all"
        },
        "loss": {
            "start_pref_e": 100,
            "limit_pref_e": 100,
            "start_pref_f": 1,
            "limit_pref_f": 1,
            "start_pref_v": 0.9,
            "limit_pref_v": 1.0,
            "_comment": " that's all"
        },
        "_comment": " traing controls",
        "training": {
            "systems": [],
            "stop_batch": 400000,
            "batch_size": 2,
            "seed": 1,
            "_comment": " display and restart",
            "_comment": " frequencies counted in batch",
            "disp_file": "lcurve.out",
            "disp_freq": 2000,
            "numb_test": 2,
            "save_freq": 2000,
            "save_ckpt": "model.ckpt",
            "load_ckpt": "model.ckpt",
            "disp_training": true,
            "time_training": true,
            "profiling": false,
            "profiling_file": "timeline.json",
            "_comment": "that's all"
        },
        "_comment": "that's all"
    },
    "fp_style": "vasp",
    "fp_skip_bad_box": "length_ratio:5;height_ratio:5",
    "shuffle_poscar": false,
    "fp_task_max": 100,
    # At most 100 data points are randomly picked and sent to the labeling step.
    "fp_task_min": 0,
    # All the data picked in the first iteration will be sent to relabel. 
    "fp_pp_path": "vasp_input",
    "fp_pp_files": ["POTCAR_Ag", "POTCAR_Au"],
    "fp_incar": "vasp_input/INCAR",
    "use_clusters": false,
    "labeled": false,
    "init_pick_number": 100,
    "iter_pick_number": 100,
    "e_trust_lo": 1e10,
    "e_trust_hi": 1e10,
    "f_trust_lo": 0.20,
    "f_trust_hi": 100.00,
    # Once the mode_deviation of the data is higher than f_trust_lo, it is selected as a candidate. Since all the data is selected from the original data set, which is proper for constructing a good DP model,  we do not especially need to exclude the data with bad box or atom overlap.   
    "_comment": " that's all "
}
```

## References

YiNan Wang et al 2022 Modelling Simul. Mater. Sci. Eng. 30 025003

Grimme S, Antony J, Ehrlich S and Krieg H 2010 J. Chem. Phys. 132 154104


