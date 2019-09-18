#  Foundations Atlas Tutorial

*Estimated time: x minutes*

## Introduction

This tutorial demonstrates how to make use of the features of Foundations Atlas. Note that any machine learning
job can be run in Atlas without modification. However, with minimal changes to the code we can take advantage of 
Atlas features that will enable us to launch many jobs and organize our model experiments more systematically.

This tutorial assumes that you have already installed Foundations Atlas. If you have not then...

In this tutorial we make use of the CIFAR-10 dataset. This is an image recognition dataset consisting of 60,000 32x32
RGB images. These images belong to 10 categories including: dogs, cats, boats, cars, etc...

## Enabling Atlas Features

You are provided with the following python scripts:
* driver.py: A driver script which downloads the dataset, prepares it for model training and evaluation, trains a simple 
convolutional network, then evaluates the model on the test set
* model.py: Code to implement the convolutional network

Note that this is a fairly standard implementation and runs without any modification.

To enable Atlas features, we only to need to make a few changes. Firstly import foundations in driver.py. Add the 
following line to the top of driver.py:

```python
import foundations
```

In model.py, there are some lines to print the test metrics. We'll replace those print
statements with calls to the function foundations.log_metric(). This function takes two arguments, a key and a value. Once a 
job successfully completes, logged metrics for each job will be visible from the Foundations GUI. Copy the following two lines
and replace the two print statements with them:

```python
foundations.log_metric('test_loss', float(scores[0]))
foundations.log_metric('test_accuracy:', float(scores[1]))
```   

Lastly, a configuration file needs to be added. Create a new file in the project directory called job.config.yaml with 
the following contents:

```yaml
project_name: 'cifar-demo'
log_level: INFO
```

## Running a Job

Activate the environment in which you have foundations installed, then from inside the project directory (cifar-demo)
run the following command:

```shell script
foundations submit scheduler . driver.py
```

This will schedule a job to be run. Now open the Atlas GUI in your browser: http://localhost:5555/projects. Click into 
the project 'cifar-demo', then you'll see the running job. Once it completes, it will have a green status and you will 
see your logged metrics.

## Running a Hyperparameter Search

Atlas makes running and tracking the results of a hyperparameter easy. Create a new file called 
'hyperparameter_search.py' and paste in the following code:

```python
import os
os.environ['FOUNDATIONS_COMMAND_LINE'] = 'True'
import foundations
import numpy as np
import copy


class SearchSpace:

    def __init__(self, min, max, type):
        self.min = min
        self.max = max
        self.type = type

    def sample(self):
        if self.type == int:
            return np.random.randint(self.min, self.max)
        elif self.type == float:
            return round(np.random.uniform(self.min, self.max), 2)


def sample_hyperparameters(hyperparameter_ranges):
    hyperparameters = copy.deepcopy(hyperparameter_ranges)
    for hparam in hyperparameter_ranges:
        if isinstance(hyperparameter_ranges[hparam], SearchSpace):
            search_space = hyperparameter_ranges[hparam]
            hyperparameters[hparam] = search_space.sample()
        elif isinstance(hyperparameter_ranges[hparam], list):
            for i, block in enumerate(hyperparameter_ranges[hparam]):
                for block_hparam in block:
                    if isinstance(block[block_hparam], SearchSpace):
                        search_space = block[block_hparam]
                        hyperparameters[hparam][i][block_hparam] = search_space.sample()
    return hyperparameters


hyperparameter_ranges = {'num_epochs': 5,
                         'batch_size': 64,
                         'learning_rate': 0.001,
                         'depthwise_separable_blocks': [{'depthwise_conv_stride': 2, 'pointwise_conv_output_filters': 6},
                                                  {'depthwise_conv_stride': 2, 'pointwise_conv_output_filters': 12}],
                         'dense_blocks': [{'size': SearchSpace(64, 256, int),
                                           'dropout_rate': SearchSpace(0.1, 0.5, float)}],
                         'decay': 1e-6}

num_jobs = 5
for _ in range(num_jobs):
    hyperparameters = sample_hyperparameters(hyperparameter_ranges)
    foundations.submit(scheduler_config='scheduler', job_dir='.', command='driver.py', params=hyperparameters, stream_job_logs=True)
```

This script samples hyperparameters uniformly from pre-defined ranges, then submits jobs using those hyperparameters. 
The job execution code is still coming from driver.py. In order to get this to work, a small modification needs to be 
made to driver.py. In the code block where the hyperparameters are defined (indicated by the comment 'define 
hyperparameters'), we'll load the sampled hyperparameters instead of defining a fixed set of hyperparameters explictely.

Replace that block with the following:

```python
# define hyperparameters
hyperparameters = foundations.load_parameters()
```

Now, to run the hyperparameter search, from the project directory (cifar-demo) simply run 

```shell script
python hyperparameter_search.py
```






