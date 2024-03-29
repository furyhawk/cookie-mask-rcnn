# Batch Inferencing

Some problem statements do not warrant the deployment of an API server
but instead methods for conducting batched inferencing where a batch
of data is provided to a script and the script churns out
a set of predictions, perhaps exported to a file.

This template provides a Python script (`src/batch_inferencing.py`)
as well as an accompanying
Dockerfile
(`docker/cookie-mask-rcnn-batch-inferencing.Dockerfile`)
for a containerised execution.

Let's first download some data for us to conduct batch inferencing on:

=== "Local Machine"

    ```bash
    $ mkdir data/batched-input-data
    $ cd data/batched-input-data
    $ gsutil -m rsync -r gs://aisg-mlops-pub-data/aclImdb_v1/detached/unsup .
    ```

To execute the script locally:

=== "Linux/macOS"

    ```bash
    # Navigate back to root directory
    $ cd ../..
    $ export PRED_MODEL_UUID="<MLFLOW_EXPERIMENT_UUID>"
    $ export PRED_MODEL_PATH="$PWD/models/$PRED_MODEL_UUID/artifacts/model/data/model"
    $ conda activate cookie-mask-rcnn
    $ python src/batch_inferencing.py \
        inference.model_path=$PRED_MODEL_PATH \
        inference.input_data_dir="$PWD/data/batched-input-data"
    ```

=== "Windows PowerShell"

    ```powershell
    # Navigate back to root directory
    $ cd ..\..
    $ $Env:PRED_MODEL_UUID='<MLFLOW_EXPERIMENT_UUID>'
    $ $Env:PRED_MODEL_PATH="$(Get-Location)\models\$Env:PRED_MODEL_UUID\artifacts\model\data\model"
    $ conda activate cookie-mask-rcnn
    $ python src/batch_inferencing.py `
        inference.model_path="$Env:PRED_MODEL_PATH" `
        inference.input_data_dir="$(Get-Location)\data\batched-input-data"
    ```

The parameter `inference.input_data_dir` assumes a directory
containing `.txt` files containing movie reviews. At the end of the
execution, __the script will log to the terminal the location of the
`.jsonl` file (`batch-infer-res.jsonl`) containing predictions__ that
look like such:

```jsonl
...
{"time": "2022-01-06T06:40:27+0000", "filepath": "/home/aisg/cookie-mask-rcnn/data/1131_2.txt", "logit_prob": 0.006387829780578613, "sentiment": "negative"}
{"time": "2022-01-06T06:40:27+0000", "filepath": "/home/aisg/cookie-mask-rcnn/data/11020_3.txt", "logit_prob": 0.0041103363037109375, "sentiment": "negative"}
{"time": "2022-01-06T06:40:27+0000", "filepath": "/home/aisg/cookie-mask-rcnn/data/11916_3.txt", "logit_prob": 0.023626357316970825, "sentiment": "negative"}
{"time": "2022-01-06T06:40:27+0000", "filepath": "/home/aisg/cookie-mask-rcnn/data/3129_2.txt", "logit_prob": 0.00018364191055297852, "sentiment": "negative"}
{"time": "2022-01-06T06:40:27+0000", "filepath": "/home/aisg/cookie-mask-rcnn/data/2444_4.txt", "logit_prob": 3.255962656112388e-05, "sentiment": "negative"}
...
```

The results are exported to a subdirectory within the
`outputs` folder. See
[here](https://hydra.cc/docs/tutorials/basic/running_your_app/working_directory/)
for more information on outputs generated by Hydra.

To use the Docker image, first build it:

=== "Linux/macOS"

    ```bash
    $ export PRED_MODEL_UUID="<MLFLOW_EXPERIMENT_UUID>"
    $ docker build \
        -t asia.gcr.io/cvhub-312105/teck/batch-inference:0.1.0 \
        --build-arg PRED_MODEL_UUID="$PRED_MODEL_UUID" \
        -f docker/cookie-mask-rcnn-batch-inferencing.Dockerfile \
        --platform linux/amd64 .
    ```

=== "Windows PowerShell"

    ```powershell
    $ $Env:PRED_MODEL_UUID='<MLFLOW_EXPERIMENT_UUID>'
    $ docker build `
        -t asia.gcr.io/cvhub-312105/teck/batch-inference:0.1.0 `
        --build-arg PRED_MODEL_UUID="$Env:PRED_MODEL_UUID" `
        -f docker/cookie-mask-rcnn-batch-inferencing.Dockerfile `
        --platform linux/amd64 .
    ```

Similar to how the predictive models are defined for the
[FastAPI servers' images](./08-deployment.md#model-serving-fastapi),
`PRED_MODEL_UUID` requires the unique ID associated
with the MLflow run that generated the predictive model that you wish
to make use of for the batch inferencing.

After building the image, you can run the container like so:

=== "Linux/macOS"

    ```bash
    $ sudo chgrp -R 2222 outputs
    $ docker run --rm \
        --env GOOGLE_APPLICATION_CREDENTIALS=/var/secret/cloud.google.com/gcp-service-account.json \
        --env INPUT_DATA_DIR=/home/aisg/cookie-mask-rcnn/data \
        -v <ABSOLUTE_PATH_TO_SA_JSON_FILE>:/var/secret/cloud.google.com/gcp-service-account.json \
        -v $PWD/models:/home/aisg/from-gcs \
        -v $PWD/outputs:/home/aisg/cookie-mask-rcnn/outputs \
        -v $PWD/data/batched-input-data:/home/aisg/cookie-mask-rcnn/data \
        asia.gcr.io/cvhub-312105/teck/batch-inference:0.1.0
    ```

=== "Windows PowerShell"

    ```powershell
    $ docker run --rm `
        --env GOOGLE_APPLICATION_CREDENTIALS=/var/secret/cloud.google.com/gcp-service-account.json `
        --env INPUT_DATA_DIR=/home/aisg/cookie-mask-rcnn/data `
        -v "<ABSOLUTE_PATH_TO_SA_JSON_FILE>:/var/secret/cloud.google.com/gcp-service-account.json" `
        -v "$(Get-Location)\models:/home/aisg/from-gcs" `
        -v "$(Get-Location)\outputs:/home/aisg/cookie-mask-rcnn/outputs" `
        -v "$(Get-Location)\data\batched-input-data:/home/aisg/cookie-mask-rcnn/data" `
        asia.gcr.io/cvhub-312105/teck/batch-inference:0.1.0
    ```

In the `docker run` command above we are passing two variables:
`GOOGLE_APPLICATION_CREDENTIALS` and `INPUT_DATA_DIR`.
The former allows the container's entrypoint to download the
predictive model specified from GCS when the container starts.
The latter
will be fed to the script's parameter: `inference.input_data_dir`.
4 volumes are attached to the container for persistence as well as
usage of host files and directories.

- `-v <PATH_TO_SA_JSON_FILE>:/var/secret/cloud.google.com/gcp-service-account.json`:
  This attaches the JSON file for the service account credentials to
  the Docker container.
- `-v $PWD/models:/home/aisg/from-gcs`: The models downloaded to the
  host machine can be used by the container after being mounted to
  `/home/aisg/from-gcs`.
- `-v $PWD/outputs:/home/aisg/cookie-mask-rcnn/outputs`:
  This is for persisting the batch inferencing outputs to the outputs
  folder on the host machine.
- `-v <PATH_TO_DIR_CONTAINING_TXT_FILES>:/home/aisg/cookie-mask-rcnn/data`:
  To provide the container with access to the data that is on the host
  machine, you need to mount the directory containing the text
  files for inferencing.

__Reference(s):__

- [Docker Docs - Use volumes](https://docs.docker.com/storage/volumes/)
