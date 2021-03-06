# Predictor implementation

Which Predictor you use depends on how your model is exported:

* [TensorFlow Predictor](#tensorflow-predictor) if your model is exported as a TensorFlow `SavedModel`
* [ONNX Predictor](#onnx-predictor) if your model is exported in the ONNX format
* [Python Predictor](#python-predictor) for all other cases

## Project files

Cortex makes all files in the project directory (i.e. the directory which contains `cortex.yaml`) available for use in your Predictor implementation. Python bytecode files (`*.pyc`, `*.pyo`, `*.pyd`), files or folders that start with `.`, and the api configuration file (e.g. `cortex.yaml`) are excluded.

The following files can also be added at the root of the project's directory:

* `.cortexignore` file, which follows the same syntax and behavior as a [.gitignore file](https://git-scm.com/docs/gitignore).
* `.env` file, which exports environment variables that can be used in the predictor. Each line of this file must follow the `VARIABLE=value` format.

For example, if your directory looks like this:

```text
./my-classifier/
├── cortex.yaml
├── values.json
├── predictor.py
├── ...
└── requirements.txt
```

You can access `values.json` in your Predictor like this:

```python
import json

class PythonPredictor:
    def __init__(self, config):
        with open('values.json', 'r') as values_file:
            values = json.load(values_file)
        self.values = values
```

## Python Predictor

### Interface

```python
# initialization code and variables can be declared here in global scope

class PythonPredictor:
    def __init__(self, config, job_spec):
        """(Required) Called once during each worker initialization. Performs
        setup such as downloading/initializing the model or downloading a
        vocabulary.

        Args:
            config (required): Dictionary passed from API configuration (if
                specified) merged with configuration passed in with Job
                Submission API. If there are conflicting keys, values in
                configuration specified in Job submission takes precedence.
            job_spec (optional): Dictionary containing the following fields:
                "job_id": A unique ID for this job
                "api_name": The name of this batch API
                "config": The config that was provided in the job submission
                "workers": The number of workers for this job
                "total_batch_count": The total number of batches in this job
                "start_time": The time that this job started
        """
        pass

    def predict(self, payload, batch_id):
        """(Required) Called once per batch. Preprocesses the batch payload (if
        necessary), runs inference, postprocesses the inference output (if
        necessary), and writes the predictions to storage (i.e. S3 or a
        database, if desired).

        Args:
            payload (required): a batch (i.e. a list of one or more samples).
            batch_id (optional): uuid assigned to this batch.
        Returns:
            Nothing
        """
        pass

    def on_job_complete(self):
        """(Optional) Called once after all batches in the job have been
        processed. Performs post job completion tasks such as aggregating
        results, executing web hooks, or triggering other jobs.
        """
        pass
```

For proper separation of concerns, it is recommended to use the constructor's `config` parameter for information such as from where to download the model and initialization files, or any configurable model parameters. You define `config` in your API configuration, and it is passed through to your Predictor's constructor. The `config` parameters in the `API configuration` can be overridden by providing `config` in the job submission requests.

### Pre-installed packages

The following Python packages are pre-installed in Python Predictors and can be used in your implementations:

```text
boto3==1.14.53
cloudpickle==1.6.0
Cython==0.29.21
dill==0.3.2
fastapi==0.61.1
google-cloud-storage==1.32.0
joblib==0.16.0
Keras==2.4.3
msgpack==1.0.0
nltk==3.5
np-utils==0.5.12.1
numpy==1.19.1
opencv-python==4.4.0.42
pandas==1.1.1
Pillow==7.2.0
pyyaml==5.3.1
requests==2.24.0
scikit-image==0.17.2
scikit-learn==0.23.2
scipy==1.5.2
six==1.15.0
statsmodels==0.12.0
sympy==1.6.2
tensorflow-hub==0.9.0
tensorflow==2.3.0
torch==1.6.0
torchvision==0.7.0
xgboost==1.2.0
```

#### Inferentia-equipped APIs

The list is slightly different for inferentia-equipped APIs:

```text
boto3==1.13.7
cloudpickle==1.6.0
Cython==0.29.21
dill==0.3.1.1
fastapi==0.54.1
google-cloud-storage==1.32.0
joblib==0.16.0
msgpack==1.0.0
neuron-cc==1.0.20600.0+0.b426b885f
nltk==3.5
np-utils==0.5.12.1
numpy==1.18.2
opencv-python==4.4.0.42
pandas==1.1.1
Pillow==7.2.0
pyyaml==5.3.1
requests==2.23.0
scikit-image==0.17.2
scikit-learn==0.23.2
scipy==1.3.2
six==1.15.0
statsmodels==0.12.0
sympy==1.6.2
tensorflow==1.15.4
tensorflow-neuron==1.15.3.1.0.2043.0
torch==1.5.1
torch-neuron==1.5.1.1.0.1721.0
torchvision==0.6.1
```

<!-- CORTEX_VERSION_MINOR x3 -->
The pre-installed system packages are listed in [images/python-predictor-cpu/Dockerfile](https://github.com/cortexlabs/cortex/tree/master/images/python-predictor-cpu/Dockerfile) (for CPU), [images/python-predictor-gpu/Dockerfile](https://github.com/cortexlabs/cortex/tree/master/images/python-predictor-gpu/Dockerfile) (for GPU), or [images/python-predictor-inf/Dockerfile](https://github.com/cortexlabs/cortex/tree/master/images/python-predictor-inf/Dockerfile) (for Inferentia).

## TensorFlow Predictor

### Interface

```python
class TensorFlowPredictor:
    def __init__(self, tensorflow_client, config, job_spec):
        """(Required) Called once during each worker initialization. Performs
        setup such as downloading/initializing the model or downloading a
        vocabulary.

        Args:
            tensorflow_client (required): TensorFlow client which is used to
                make predictions. This should be saved for use in predict().
            config (required): Dictionary passed from API configuration (if
                specified) merged with configuration passed in with Job
                Submission API. If there are conflicting keys, values in
                configuration specified in Job submission takes precedence.
            job_spec (optional): Dictionary containing the following fields:
                "job_id": A unique ID for this job
                "api_name": The name of this batch API
                "config": The config that was provided in the job submission
                "workers": The number of workers for this job
                "total_batch_count": The total number of batches in this job
                "start_time": The time that this job started
        """
        self.client = tensorflow_client
        # Additional initialization may be done here

    def predict(self, payload, batch_id):
        """(Required) Called once per batch. Preprocesses the batch payload (if
        necessary), runs inference (e.g. by calling
        self.client.predict(model_input)), postprocesses the inference output
        (if necessary), and writes the predictions to storage (i.e. S3 or a
        database, if desired).

        Args:
            payload (required): a batch (i.e. a list of one or more samples).
            batch_id (optional): uuid assigned to this batch.
        Returns:
            Nothing
        """
        pass

    def on_job_complete(self):
        """(Optional) Called once after all batches in the job have been
        processed. Performs post job completion tasks such as aggregating
        results, executing web hooks, or triggering other jobs.
        """
        pass
```

<!-- CORTEX_VERSION_MINOR -->
Cortex provides a `tensorflow_client` to your Predictor's constructor. `tensorflow_client` is an instance of [TensorFlowClient](https://github.com/cortexlabs/cortex/tree/master/pkg/cortex/serve/cortex_internal/lib/client/tensorflow.py) that manages a connection to a TensorFlow Serving container to make predictions using your model. It should be saved as an instance variable in your Predictor, and your `predict()` function should call `tensorflow_client.predict()` to make an inference with your exported TensorFlow model. Preprocessing of the JSON payload and postprocessing of predictions can be implemented in your `predict()` function as well.

When multiple models are defined using the Predictor's `models` field, the `tensorflow_client.predict()` method expects a second argument `model_name` which must hold the name of the model that you want to use for inference (for example: `self.client.predict(payload, "text-generator")`).

For proper separation of concerns, it is recommended to use the constructor's `config` parameter for information such as from where to download the model and initialization files, or any configurable model parameters. You define `config` in your API configuration, and it is passed through to your Predictor's constructor. The `config` parameters in the `API configuration` can be overridden by providing `config` in the job submission requests.

### Pre-installed packages

The following Python packages are pre-installed in TensorFlow Predictors and can be used in your implementations:

```text
boto3==1.14.53
dill==0.3.2
fastapi==0.61.1
google-cloud-storage==1.32.0
msgpack==1.0.0
numpy==1.19.1
opencv-python==4.4.0.42
pyyaml==5.3.1
requests==2.24.0
tensorflow-hub==0.9.0
tensorflow-serving-api==2.3.0
tensorflow==2.3.0
```

<!-- CORTEX_VERSION_MINOR -->
The pre-installed system packages are listed in [images/tensorflow-predictor/Dockerfile](https://github.com/cortexlabs/cortex/tree/master/images/tensorflow-predictor/Dockerfile).

## ONNX Predictor

### Interface

```python
class ONNXPredictor:
    def __init__(self, onnx_client, config, job_spec):
        """(Required) Called once during each worker initialization. Performs
        setup such as downloading/initializing the model or downloading a
        vocabulary.

        Args:
            onnx_client (required): ONNX client which is used to make
                predictions. This should be saved for use in predict().
            config (required): Dictionary passed from API configuration (if
                specified) merged with configuration passed in with Job
                Submission API. If there are conflicting keys, values in
                configuration specified in Job submission takes precedence.
            job_spec (optional): Dictionary containing the following fields:
                "job_id": A unique ID for this job
                "api_name": The name of this batch API
                "config": The config that was provided in the job submission
                "workers": The number of workers for this job
                "total_batch_count": The total number of batches in this job
                "start_time": The time that this job started
        """
        self.client = onnx_client
        # Additional initialization may be done here

    def predict(self, payload, batch_id):
        """(Required) Called once per batch. Preprocesses the batch payload (if
        necessary), runs inference (e.g. by calling
        self.client.predict(model_input)), postprocesses the inference output
        (if necessary), and writes the predictions to storage (i.e. S3 or a
        database, if desired).

        Args:
            payload (required): a batch (i.e. a list of one or more samples).
            batch_id (optional): uuid assigned to this batch.
        Returns:
            Nothing
        """
        pass

    def on_job_complete(self):
        """(Optional) Called once after all batches in the job have been
        processed. Performs post job completion tasks such as aggregating
        results, executing web hooks, or triggering other jobs.
        """
        pass
```

<!-- CORTEX_VERSION_MINOR -->
Cortex provides an `onnx_client` to your Predictor's constructor. `onnx_client` is an instance of [ONNXClient](https://github.com/cortexlabs/cortex/tree/master/pkg/cortex/serve/cortex_internal/lib/client/onnx.py) that manages an ONNX Runtime session to make predictions using your model. It should be saved as an instance variable in your Predictor, and your `predict()` function should call `onnx_client.predict()` to make an inference with your exported ONNX model. Preprocessing of the JSON payload and postprocessing of predictions can be implemented in your `predict()` function as well.

When multiple models are defined using the Predictor's `models` field, the `onnx_client.predict()` method expects a second argument `model_name` which must hold the name of the model that you want to use for inference (for example: `self.client.predict(model_input, "text-generator")`).

For proper separation of concerns, it is recommended to use the constructor's `config` parameter for information such as from where to download the model and initialization files, or any configurable model parameters. You define `config` in your API configuration, and it is passed through to your Predictor's constructor. The `config` parameters in the `API configuration` can be overridden by providing `config` in the job submission requests.

### Pre-installed packages

The following Python packages are pre-installed in ONNX Predictors and can be used in your implementations:

```text
boto3==1.14.53
dill==0.3.2
fastapi==0.61.1
google-cloud-storage==1.32.0
msgpack==1.0.0
numpy==1.19.1
onnxruntime==1.4.0
pyyaml==5.3.1
requests==2.24.0
```

<!-- CORTEX_VERSION_MINOR x2 -->
The pre-installed system packages are listed in [images/onnx-predictor-cpu/Dockerfile](https://github.com/cortexlabs/cortex/tree/master/images/onnx-predictor-cpu/Dockerfile) (for CPU) or [images/onnx-predictor-gpu/Dockerfile](https://github.com/cortexlabs/cortex/tree/master/images/onnx-predictor-gpu/Dockerfile) (for GPU).
