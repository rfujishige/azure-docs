---
title: Dataset versioning
titleSuffix: Azure Machine Learning
description: Learn how to version machine learning datasets and how versioning works with machine learning pipelines.
services: machine-learning
ms.service: machine-learning
ms.subservice: mldata
ms.author: larryfr
author: blackmist
ms.date: 10/21/2021
ms.topic: how-to
ms.custom: devx-track-python, data4ml

# Customer intent: As a data scientist, I want to version and track datasets so I can use and share them across multiple machine learning experiments.
---

# Version and track Azure Machine Learning datasets

In this article, you'll learn how to version and track Azure Machine Learning datasets for reproducibility. Dataset versioning is a way to bookmark the state of your data so that you can apply a specific version of the dataset for future experiments.

Typical versioning scenarios:

* When new data is available for retraining
* When you're applying different data preparation or feature engineering approaches

## Prerequisites

For this tutorial, you need:

- [Azure Machine Learning SDK for Python installed](/python/api/overview/azure/ml/install). This SDK includes the [azureml-datasets](/python/api/azureml-core/azureml.core.dataset) package.
    
- An [Azure Machine Learning workspace](concept-workspace.md). Retrieve an existing one by running the following code, or [create a new workspace](how-to-manage-workspace.md).

    ```Python
    import azureml.core
    from azureml.core import Workspace
    
    ws = Workspace.from_config()
    ```
- An [Azure Machine Learning dataset](how-to-create-register-datasets.md).

<a name="register"></a>

## Register and retrieve dataset versions

By registering a dataset, you can version, reuse, and share it across experiments and with colleagues. You can register multiple datasets under the same name and retrieve a specific version by name and version number.

### Register a dataset version

The following code registers a new version of the `titanic_ds` dataset by setting the `create_new_version` parameter to `True`. If there's no existing `titanic_ds` dataset registered with the workspace, the code creates a new dataset with the name `titanic_ds` and sets its version to 1.

```Python
titanic_ds = titanic_ds.register(workspace = workspace,
                                 name = 'titanic_ds',
                                 description = 'titanic training data',
                                 create_new_version = True)
```

### Retrieve a dataset by name

By default, the [get_by_name()](/python/api/azureml-core/azureml.core.dataset.dataset#get-by-name-workspace--name--version--latest--) method on the `Dataset` class returns the latest version of the dataset registered with the workspace. 

The following code gets version 1 of the `titanic_ds` dataset.

```Python
from azureml.core import Dataset
# Get a dataset by name and version number
titanic_ds = Dataset.get_by_name(workspace = workspace,
                                 name = 'titanic_ds', 
                                 version = 1)
```

<a name="best-practice"></a>

## Versioning best practice

When you create a dataset version, you're *not* creating an extra copy of data with the workspace. Because datasets are references to the data in your storage service, you have a single source of truth, managed by your storage service.

>[!IMPORTANT]
> If the data referenced by your dataset is overwritten or deleted, calling a specific version of the dataset does *not* revert the change.

When you load data from a dataset, the current data content referenced by the dataset is always loaded. If you want to make sure that each dataset version is reproducible, we recommend that you not modify data content referenced by the dataset version. When new data comes in, save new data files into a separate data folder and then create a new dataset version to include data from that new folder.

The following image and sample code show the recommended way to structure your data folders and to create dataset versions that reference those folders:

![Folder structure](./media/how-to-version-track-datasets/folder-image.png)

```Python
from azureml.core import Dataset

# get the default datastore of the workspace
datastore = workspace.get_default_datastore()

# create & register weather_ds version 1 pointing to all files in the folder of week 27
datastore_path1 = [(datastore, 'Weather/week 27')]
dataset1 = Dataset.File.from_files(path=datastore_path1)
dataset1.register(workspace = workspace,
                  name = 'weather_ds',
                  description = 'weather data in week 27',
                  create_new_version = True)

# create & register weather_ds version 2 pointing to all files in the folder of week 27 and 28
datastore_path2 = [(datastore, 'Weather/week 27'), (datastore, 'Weather/week 28')]
dataset2 = Dataset.File.from_files(path = datastore_path2)
dataset2.register(workspace = workspace,
                  name = 'weather_ds',
                  description = 'weather data in week 27, 28',
                  create_new_version = True)

```

<a name="pipeline"></a>

## Version an ML pipeline output dataset

You can use a dataset as the input and output of each [ML pipeline](concept-ml-pipelines.md) step. When you rerun pipelines, the output of each pipeline step is registered as a new dataset version.

ML pipelines populate the output of each step into a new folder every time the pipeline reruns. This behavior allows the versioned output datasets to be reproducible. Learn more about [datasets in pipelines](./how-to-create-machine-learning-pipelines.md#steps).

```Python
from azureml.core import Dataset
from azureml.pipeline.steps import PythonScriptStep
from azureml.pipeline.core import Pipeline, PipelineData
from azureml.core. runconfig import CondaDependencies, RunConfiguration

# get input dataset 
input_ds = Dataset.get_by_name(workspace, 'weather_ds')

# register pipeline output as dataset
output_ds = PipelineData('prepared_weather_ds', datastore=datastore).as_dataset()
output_ds = output_ds.register(name='prepared_weather_ds', create_new_version=True)

conda = CondaDependencies.create(
    pip_packages=['azureml-defaults', 'azureml-dataprep[fuse,pandas]'], 
    pin_sdk_version=False)

run_config = RunConfiguration()
run_config.environment.docker.enabled = True
run_config.environment.python.conda_dependencies = conda

# configure pipeline step to use dataset as the input and output
prep_step = PythonScriptStep(script_name="prepare.py",
                             inputs=[input_ds.as_named_input('weather_ds')],
                             outputs=[output_ds],
                             runconfig=run_config,
                             compute_target=compute_target,
                             source_directory=project_folder)
```

<a name="track"></a>

## Track data in your experiments

Azure Machine Learning tracks your data throughout your experiment as input and output datasets.  

The following are scenarios where your data is tracked as an **input dataset**. 

* As a `DatasetConsumptionConfig` object through either the `inputs` or `arguments` parameter of your `ScriptRunConfig` object when submitting the experiment run. 

* When methods like, get_by_name() or get_by_id() are called in your script. For this scenario, the name assigned to the dataset when you registered it to the workspace is the name displayed. 

The following are scenarios where your data is tracked as an **output dataset**.  

* Pass an `OutputFileDatasetConfig` object through either the `outputs` or `arguments` parameter when submitting an experiment run. `OutputFileDatasetConfig` objects can also be used to persist data between pipeline steps. See [Move data between ML pipeline steps.](how-to-move-data-in-out-of-pipelines.md)
  
* Register a dataset in your script. For this scenario, the name assigned to the dataset when you registered it to the workspace is the name displayed. In the following example, `training_ds` is the name that would be displayed.

    ```Python
   training_ds = unregistered_ds.register(workspace = workspace,
                                     name = 'training_ds',
                                     description = 'training data'
                                     )
    ```

* Submit child run with an unregistered dataset in script. This results in an anonymous saved dataset.

### Trace datasets in experiment runs

For each Machine Learning experiment, you can easily trace the datasets used as input with the experiment `Run` object.

The following code uses the [`get_details()`](/python/api/azureml-core/azureml.core.run.run#get-details--) method to track which input datasets were used with the experiment run:

```Python
# get input datasets
inputs = run.get_details()['inputDatasets']
input_dataset = inputs[0]['dataset']

# list the files referenced by input_dataset
input_dataset.to_path()
```

You can also find the `input_datasets` from experiments by using the [Azure Machine Learning studio](). 

The following image shows where to find the input dataset of an experiment on Azure Machine Learning studio. For this example, go to your **Experiments** pane and open the **Properties** tab for a specific run of your experiment, `keras-mnist`.

![Input datasets](./media/how-to-version-track-datasets/input-datasets.png)

Use the following code to register models with datasets:

```Python
model = run.register_model(model_name='keras-mlp-mnist',
                           model_path=model_path,
                           datasets =[('training data',train_dataset)])
```

After registration, you can see the list of models registered with the dataset by using Python or go to the [studio](https://ml.azure.com/).

The following view is from the **Datasets** pane under **Assets**. Select the dataset and then select the **Models** tab for a list of the models that are registered with the dataset. 

![Input datasets models](./media/how-to-version-track-datasets/dataset-models.png)

## Next steps

* [Train with datasets](how-to-train-with-datasets.md)
* [More sample dataset notebooks](https://github.com/Azure/MachineLearningNotebooks/tree/master/how-to-use-azureml/work-with-data/)