# Object Detection Tutorial

This tutorial introduces some Raster Vision concepts and demonstrates how to run an object detection workflow.

## Raster Vision concepts

### Commands

Raster Vision supports four *commands*.
* `process_training_data`: creates training chips from raster data and annotations and converts it to backend-specific format
* `train`: trains a model
* `predict`: makes predictions on raster data using a model
* `eval`: evaluates the quality of the predictions against ground truth annotations

### Config files
Each command is configured with a *command config* file in the form of a JSON-formatted [Protocol Buffer](https://developers.google.com/protocol-buffers/docs/pythontutorial), and there is a different schema for each command. The advantage of using protobufs (rather than conventional JSON files) is that we get compositional schemas, validation, and automatically generated Python clients for free. The schemas are stored in the `.proto` files in [rv2.protos](../src/rv2/protos), and are compiled to Python classes using `./scripts/compile`.

### Projects

Data is organized into a set of *training projects* and a set of *test projects*. Each training project consists of a *raster source* with imagery covering some area of interest, and a ground truth *annotation source* with labels generated by people. Each test project additionally contains a prediction annotation source where predictions are stored.

### Machine learning tasks and backends

A *machine learning task* implements functionality specific to an abstract prediction problem such as object detection. In contrast, a *machine learning backend* implements a bridge to a (usually third party) concrete implementation of that task, such as the [Tensorflow Object Detection API](https://github.com/tensorflow/models/tree/master/research/object_detection).

Raster Vision is designed to be easy to extend to new raster and annotation sources, and machine learning tasks and backends.

### Workflows

A *workflow* is a network of dependent commands. For instance, you might want to make two variants of a dataset, then train a set of model on each dataset (representing a hyperparameter sweep), make predictions for each model on a test set, and then evaluate the quality of the predictions. This workflow can be represented as a tree of commands. At the moment, we have only implemented a *chain workflow*, which has the structure of a linked list or chain, and simply runs the four commands in sequence. A workflow is configured by a *workflow config* file, and executing it generates a set of command config files. By decoupling command and workflow configs, Raster Vision is designed to make it easy to implement new, more complex workflows.

## File hierarchy

When using chain workflows, you will need to use the following file hierarchy convention for the output of Raster Vision since the workflow runner generates URIs according to this convention. However, if you would like to manually write configs and run commands outside of a workflow (for instance, when making new predictions for an existing model), then you do not need to follow any particular convention since absolute paths are used in the command config files.

```
# The root of the file hierarchy -- local or on S3
<RVROOT>
# Output of running Raster Vision
<RVROOT>/rv-output

# Output and config file for a run of process_training_data command
<RVROOT>/rv-output/datasets/<DATASET_KEY>/output/
<RVROOT>/rv-output/datasets/<DATASET_KEY>/config.json

# Output (and config file) for a run of train command
<RVROOT>/rv-output/datasets/<DATASET_KEY>/models/<MODEL_KEY>/output/
<RVROOT>/rv-output/datasets/<DATASET_KEY>/models/<MODEL_KEY>/config.json

# Output (and config file) for a run of predict command
<RVROOT>/rv-output/datasets/<DATASET_KEY>/models/<MODEL_KEY>/predictions/<PREDICTION_KEY>/output/
<RVROOT>/rv-output/datasets/<DATASET_KEY>/models/<MODEL_KEY>/predictions/<PREDICTION_KEY>/config.json

# Output (and config file) for a run of eval command
<RVROOT>/rv-output/datasets/<DATASET_KEY>/models/<MODEL_KEY>/predictions/<PREDICTION_KEY>/evals/<EVAL_KEY>/output/
<RVROOT>/rv-output/datasets/<DATASET_KEY>/models/<MODEL_KEY>/predictions/<PREDICTION_KEY>/evals/<EVAL_KEY>/config.json
```

This convention is convenient and intuitive because it stores the config file for a run of a command adjacent to its output, which makes it easy to reason about how output files were generated. It also nests models inside datasets, predictions inside models, and evals inside predictions, which mirrors the one-to-many relationship in arbitrary workflows between datasets and models, models and predictions, and predictions and evals. We also like to use the following convention for other files that are input to Raster Vision, but this isn't required.

```
# Pretrained model files provided by ML backend
<RVROOT>/pretrained-models
# Workflow config files
<RVROOT>/workflow-configs
# Config files in format specified by ML backend
<RVROOT>/backend-configs
# Data that is derivative of raw data
<RVROOT>/processed-data
```

## Setup

Before running any commands, you will need to download some data.
* Download the Mobilenet [pretrained model](http://download.tensorflow.org/models/object_detection/ssd_mobilenet_v1_coco_2017_11_17.tar.gz) and move the file to `<RVROOT>/pretrained-models/ssd_mobilenet_v1_coco_2017_11_17.tar.gz`.
* Download the [ISPRS Potsdam](http://www2.isprs.org/commissions/comm3/wg4/2d-sem-label-potsdam.html) imagery using the [data request form](http://www2.isprs.org/commissions/comm3/wg4/data-request-form2.html) and place it in `<RVROOT>/raw-data/isprs-potsdam`.
* Download the [cowc-potsdam annotations](data/cowc-potsdam-annotations.zip) and place them in `<RVROOT>/processed-data/cowc-potsdom/annotations/`. These files were generated from the [COWC car detection dataset](https://gdo152.llnl.gov/cowc/) using scripts in [rv2.utils.cowc](../src/rv2/utils/cowc/).

Next, you will need to copy and edit some config files.
* Copy the Mobilenet [backend config files](../src/rv2/samples/backend-configs/tf-object-detection-api/) to `<RVROOT>/backend-configs/tf-object-detection-api/`.
* Copy the [workflow config files](../src/rv2/samples/workflow-configs/) to `<RVROOT>/workflow-configs/`. These configs contain URI schemas which are strings containing parameters (eg. `{rv_root}`) which are expanded into absolute URIs when the workflow is executed. The `local_uri_map` and `remote_uri_map` fields define the value of the parameters for the two execution environments. This makes it easy to switch between local and remote execution. You will need to update the values of these maps for your own environment.

## Run test workflow locally

After everything is setup, it is wise to run a sample workflow (with a smaller dataset and only a few training steps) locally to make sure everything works. To do this, run the Docker container locally using
```
./scripts/run --cpu
```
Then compile the files in `src/rv2/proto/*.proto` into Python files by running the following. This only needs to be done once.
```
./scripts/compile
```
Finally, generate the individual command config files and run the commands locally using
```
python -m rv2.utils.chain_workflow \
    <RVROOT>/workflow-configs/cowc-potsdam-sample.json \
    --run
```
This should result in a hierarchy of files in `<RVROOT>/rv-output` which includes the generated config files. You can run a command manually on one of the generated config files, in this case for prediction, using the following. The particular keys can be found in the workflow config file.
```
python -m rv2.run predict \    
    <RVROOT>/rv-output/datasets/<DATASET_KEY>/models/<MODEL_KEY>/predictions/<PREDICTION_KEY>/config.json
```

If for some reason you want to re-run a subset of the commands in the workflow, for instance `predict` and `eval`, you can do so with
```
python -m rv2.utils.chain_workflow \
    <RVROOT>/workflow-configs/cowc-potsdam-sample.json \
    predict eval \
    --run
```

## Run full workflow remotely

Since it will take a long time to run the full workflow, it probably makes sense to run it remotely on AWS Batch. Before doing so, make sure all data and config files are copied to the right locations (as defined by the `remote_uri_map` field in the workflow config) on S3. Then run
```
python -m rv2.utils.chain_workflow \
    <RVROOT>/workflow-configs/cowc-potsdam.json \
    --run --remote --branch <REMOTE_GIT_BRANCH>
```
While the `train` command is running, you can view Tensorboard at `http://<ec2_public_dns>:6006`.

If you don't use AWS and have a local GPU, you can start the GPU container and run it locally with
```
./scripts/run --gpu
python -m rv2.utils.chain_workflow \
    <RVROOT>/workflow-configs/cowc-potsdam.json \
    --run
```

## Expected results

Running the workflow on AWS Batch (using C4 for CPU and P2 for GPU jobs) should take around 12 hours. In the output file generated by the `eval` command, the F1 score should be around 0.85 and the predictions (visualized in QGIS) should look something like this. These results were obtained from training with 1500 labeled car instances for 150,000 steps. It's very likely that better results can be obtained with fewer steps -- more experimentation is needed.

![Predictions on COWC Potsdam dataset](img/cowc-potsdam-predictions.png)