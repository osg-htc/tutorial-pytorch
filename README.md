---
ospool:
  path: software_examples/ai/tutorial-pytorch/README.md
---

The OSPool can be used as a platform to carry out machine learning and artificial intelligence research. 
The following tutorial uses the common machine learning framework, PyTorch.

## Using PyTorch on OSPool
The preferred method of using a software on the the OSPool is to use a [container.](https://portal.osg-htc.org/documentation/htc_workloads/using_software/containers/) The guide shows how to run PyTorch by downloading our desired version of PyTorch images from DockerHub.

### Pulling an Image from Docker
**Please note that the `docker build` will not work on the access point**. Apptainer is installed on the access point and users can use Apptainer to either build an image from the definition file or use `apptainer pull` to create a `.sif` file from Docker images. At the time the guide is written, the latest version of PyTorch is 2.9.0. Before pulling the image/software from Docker it is a good practice to set up the cache directory of Apptainer. Run the following command on the command prompt

```
[user@ap]$ mkdir $HOME/tmp
[user@ap]$ export TMPDIR=$HOME/tmp
[user@ap]$ export APPTAINER_TMPDIR=$HOME/tmp
[user@ap]$ export APPTAINER_CACHEDIR=$HOME/tmp
```

Now, we pull the image and convert it to a `.sif` file using `apptainer pull`

```
[user@ap]$ apptainer pull  pytorch-2.9.0.sif docker://pytorch/pytorch:2.9.0-cuda12.6-cudnn9-runtime
```

#### Transfer the image using OSDF
The above command will create a singularity container named `pytorch-2.9.0.sif` in your current directory. The image will be reused for each job, and thus the preferred transfer method is [OSDF](https://portal.osg-htc.org/documentation/htc_workloads/managing_data/osdf/). Store the `pytorch-2.9.0.sif` file your data directory on the access point (see table [here](https://portal.osg-htc.org/documentation/htc_workloads/managing_data/overview/)), and then use the OSDF url directly in the `container_image` attribute.  Note that you can not use shell variable expansion in the submit file - be sure to replace the access point and username with your actual OSPool username.

```
container_image = osdf:///ospool/<AP/data/<USERNAME>/pytorch-2.9.0.sif
<other usual submit file lines> 
queue
```

## Running an ML job using PyTorch
For this tutorial, we will see how to use PyTorch to run a machine learning workflow from the MNIST database. To download the materials for this tutorial, use the command

```
git clone https://github.com/OSGConnect/tutorial-pytorch
```

The github repository contains a tarball of the MNIST data-`MNIST_data.tar.gz`, a wrapper script- `pytorch_cnn.sh` that untars the data and runs the python script-`main.py` to train a neural network on this MNIST database. The content of the `pytorch_cnn.sh` wrapper script is given below:

```
#!/bin/bash

set -e

echo "Hello OSPool from Job $1 running on `hostname`"

# untar the test and training data
tar zxf MNIST_data.tar.gz

# run the PyTorch model
python main.py --save-model --epochs 20

# remove the data directory
rm -r data
```

A submit script-`pytorch_cnn.sub` is also there to submit the PyTorch job on the OSPool using the container that is provided by OSG. The contents of `pytorch_cnn.sub` file are:

```
container_image = osdf:///ospool/<AP/data/<USERNAME>/pytorch-2.9.0.sif

log = logs/pytorch_cnn.log
error = logs/pytorch_cnn.err
output = output/pytorch_cnn.out

executable = pytorch_cnn.sh
arguments = $(Process)

# Transfer the python script and the MNIST database to the compute node
transfer_input_files = main.py, MNIST_data.tar.gz

# We require a machine with a compatible version of the CUDA driver
require_gpus = (DriverVersion >= 10.1)

# We must request 1 CPU in addition to 1 GPU
request_cpus = 1
request_gpus = 1

request_memory = 3GB
request_disk = 5GB

queue 1
```


### Create Log Directories and Submit Job
You will need to create the `logs` and `output` directories to hold the files that will be created for each job. You can create both directories at once with the command

```
mkdir logs output
```

Submit the job using 

```
condor_submit pytorch_cnn.sub
```

### Output 
The output of the code will be the CNN Network that was trained. It will be returned to us as a file
`mnist_cnn.pt`. The are also some output stats
on the training and test error in the `pytorch_cnn.out` file 

```
Test set: Average loss: 0.0278, Accuracy: 9909/10000 (99%)
```

