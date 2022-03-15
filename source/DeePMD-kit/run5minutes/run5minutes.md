
# How to Setup a DeePMD-kit Training within 5 Minutes

DeePMD-kit is a software to implement Deep Potential. There is a lot of information on the Internet, but there are not so many tutorials for the new hand, and the official guide is too long. Today, I'll take you 5 minutes to get started with DeePMD-kit. 

Let's take a look at the training process of DeePMD-kit:

Prepare data -->  Training --> Freeze the model

What? Only three steps? Yes, it's that simple.<!--more--> 

1. ***Preparing data*** is converting the computational results of DFT to data that can be recognized by the DeePMD-kit. 
2. ***Training*** is train a Deep Potential model using the DeePMD-kit with data prepared in the previous step. 
3. Finally, what we need to do is to ***freeze the restart file in the training process into a model***, in other words is to extract the neural network parameters into a file for subsequent use. I believe you can't wait to get started. Let's go!

## 1. Preparing Data

The data format of the DeePMD-kit is introduced in the [official document](https://deepmd.readthedocs.io/) but seems complex. Don't worry, I'd like to introduce a data processing tool: ***dpdata***! You can use only one line Python scripts to process data. So easy!

```py
import dpdata
dpdata.LabeledSystem('OUTCAR').to('deepmd/npy', 'data', set_size=200)
```

In this example, we converted the computational results of the VASP in the `OUTCAR` to the data format of the DeePMD-kit and saved in to a directory named `data`, where `npy` is the compressed format of the numpy, which is required by the DeePMD-kit training. We assume `OUTCAR` stores 1000 frames of molecular dynamics trajectory, then where will be 1000 points after converting. `set_size=200` means these 1000 points will be divided into 5 subsets, which is named as `data/set.000`\~`data/set.004`, respectively. The size of each set is 200. In these 5 sets, `data/set.000`\~`data/set.003` will be considered as the training set by the DeePMD-kit, and `data/set.004` will be considered as the test set. The last set will be considered as the test set by the DeePMD-kit by default. If there is only one set, the set will be both the training set and the test set. (Of course, such test set is meaningless.) 

## 2. Training

It's required to prepare an input script to start the DeePMD-kit training. Are you still out of the fear of being dominated by INCAR script?  Don't worry, it's much easier to configure the DeePMD-kit than configuring the VASP. First, let's download an example and save to `input.json`:

```sh
wget https://raw.githubusercontent.com/deepmodeling/deepmd-kit/v1.3.3/examples/water/train/water_se_a.json -O input.json
```

The strength of the DeePMD-kit is that the same training parameters are suitable for different systems, so we only need to slightly modify `input.json` to start training. Here is the first parameter to modify:

```json
"type_map":     ["O", "H"],
```

In the DeePMD-kit data, each atom type is numbered as an integer starting from 0. The parameter gives an element name to each atom in the numbering system. Here, we can copy from the content of `data/type_map.raw`. For example,

```json
"type_map":    ["A", "B","C"],
```

Next, we are going to modify the neighbour searching parameter:

```json
"sel":       [46, 92],
```

Each number in this list gives the maximum number of atoms of each type among neighbor atoms of an atom. For example, `46` means there are at most 46 `O` (type `0`) neighbours. Here, our elements were modified to `A`, `B`, and `C`, so this parameters is also required to modify. What to do if you don't know the maximum number of neighbors? You can be roughly estimate one by the density of the system, or try a number blindly. If it is not big enough, the DeePMD-kit will shoot WARNINGS.  Below we changed it to 

```json
"sel":       [64, 64, 64]
```

In addtion, we need to modify

```
"systems":     ["../data/"],
```

to

```
"systems":     ["./data/"],
```

It is because that the directory to write to is `./data/` in the current directory. Here I'd like to introduce the definition of the data system. The DeePMD-kit considers that data with corresponding element types and atomic numbers form a system. Our data is generated from a molecular dynamics simulation and meets this condition, so we can put them into one system. Dpdata works the same way. If data cannot be put into a system, multiple systems is required to be set as a list here:
```json
 "training": {
        "systems": ["system1", "system2"]
```

Finnally, we are likely to modify another two parameters:

```json
"stop_batch":   1000000,
"batch_size":   1,
```
`stop_batch` is the numebr of training step using the SGD method of deep learning, and `batch_size` is the mini-batch size of data in each step.
If we want to reduce `stop_batch` and use `batch_size` that the DeePMD-kit recommends, we can use

```json
"stop_batch":   500000,
"batch_size":   "auto",
```

Now we have succesfully set a input file! To start training, we execuate

```sh
dp train input.json
```

and wait for results. During the training process, we can see `lcurve.out` to observe the error reduction.Among them, Column 4 and 5 are the test and training errors of energy (normalized by the number of atoms), and Column 6 and 7 are the test and training errors of the force. 

## 3. Freeze the Model

After training, we can use the following script to freeze the model:

```sh
dp freeze
```

The default filename of the output model is `frozen_model.pb`. As so, we have got a good or bad DP model. As for the reliability of this model and how to use it, I will give you a detailed tutorial in the next post.
