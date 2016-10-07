tensorflow on CloudML
====

refs: https://cloud.google.com/ml/docs/quickstarts/training

CloudML Setup
-------------

refs: https://cloud.google.com/ml/docs/how-tos/getting-set-up

install miniconda: http://conda.pydata.org/miniconda.html

install CloudML SDK

```
conda create --name cloudml python=2.7
source activate cloudml
pip install -r requirements.txt
pip install --upgrade --ignore-installed setuptools \
  https://storage.googleapis.com/tensorflow/mac/cpu/tensorflow-0.11.0rc0-py2-none-any.whl
gcloud components install beta
gcloud beta auth application-default login
pip install --upgrade --force-reinstall \
  https://storage.googleapis.com/cloud-ml/sdk/cloudml.latest.tar.gz
```

verify your envroiment

```
curl https://storage.googleapis.com/cloud-ml/scripts/check_environment.py | python
```

training local
-------------

```
cd trainable
```
Clear the output from any previous local run.
Train locally.
```
rm -rf data/
python -m trainer.task
```


Inspect job
```
tensorboard --logdir=data/ --port=8080
open http://localhost:8080
```

CloudML run single worker
----

```
cd trainable
```

setup

```
JOB_NAME=mnist_1
PROJECT_ID=`gcloud config list project --format "value(core.project)"`
TRAIN_BUCKET=gs://${PROJECT_ID}-ml
TRAIN_PATH=${TRAIN_BUCKET}/${JOB_NAME}
gsutil rm -rf ${TRAIN_PATH}
```

run

```
gcloud beta ml jobs submit training ${JOB_NAME} \
  --package-path=trainer \
  --module-name=trainer.task \
  --staging-bucket="${TRAIN_BUCKET}" \
  --region=us-central1 \
  -- \
  --train_dir="${TRAIN_PATH}/train"
```

waiting finish jobs

```
gcloud beta ml jobs describe --project ${PROJECT_ID} ${JOB_NAME}
```

CloudML run distributed multiple workers
----

```
cd distributed
```

### training local

```
rm -rf output/
python -m trainer.task \
  --train_data_paths=gs://cloud-ml-data/mnist/train.tfr.gz \
  --eval_data_paths=gs://cloud-ml-data/mnist/eval.tfr.gz \
  --output_path=output
```

### training CloudML

setup
```
JOB_NAME=distributed_1
PROJECT_ID=`gcloud config list project --format "value(core.project)"`
TRAIN_BUCKET=gs://${PROJECT_ID}-ml
TRAIN_PATH=${TRAIN_BUCKET}/${JOB_NAME}
gsutil rm -rf ${TRAIN_PATH}
```

```
cat << EOF > config.yaml
trainingInput:
  scaleTier: STANDARD_1
EOF
```

submit jobs

```
gcloud beta ml jobs submit training ${JOB_NAME} \
  --package-path=trainer \
  --module-name=trainer.task \
  --staging-bucket="${TRAIN_BUCKET}" \
  --region=us-central1 \
  --config=config.yaml \
  -- \
  --train_data_paths="gs://cloud-ml-data/mnist/train.tfr.gz" \
  --eval_data_paths="gs://cloud-ml-data/mnist/eval.tfr.gz" \
  --output_path="${TRAIN_PATH}/output"
```

Inspect job

```
gcloud beta ml jobs describe --project ${PROJECT_ID} ${JOB_NAME}
```

hyperparameter tuning
---------------------

```
cd hptuning
```

setup

```
JOB_NAME=mnist_hptuning_1
PROJECT_ID=`gcloud config list project --format "value(core.project)"`
TRAIN_BUCKET=gs://${PROJECT_ID}-ml
TRAIN_PATH=${TRAIN_BUCKET}/${JOB_NAME}
gsutil rm -rf ${TRAIN_PATH}
```

```
cat << EOF > config.yaml
trainingInput:
  # Use a cluster with many workers and a few parameter servers.
  scaleTier: STANDARD_1
  # Hyperparameter-tuning specification.
  hyperparameters:
    # Maximize the objective value.
    goal: MAXIMIZE
    # Run at most 10 trials with different hyperparameters.
    maxTrials: 10
    # Run two trials at a time.
    maxParallelTrials: 2
    params:
      # Allow the size of the first hidden layer to vary between 40 and 400.
      # One value in this range will be passed to each trial via the
      # --hidden1 command-line flag.
      - parameterName: hidden1
        type: INTEGER
        minValue: 40
        maxValue: 400
        scaleType: UNIT_LINEAR_SCALE
      # Allow the size of the second hidden layer to vary between 5 and 250.
      # One value in this range will be passed to each trial via the
      # --hidden2 command-line flag.
      - parameterName: hidden2
        type: INTEGER
        minValue: 5
        maxValue: 250
        scaleType: UNIT_LINEAR_SCALE
      # Allow the learning rate to vary between 0.0001 and 0.5.
      # One value in this range will be passed to each trial via the
      # --learning_rate command-line flag.
      - parameterName: learning_rate
        type: DOUBLE
        minValue: 0.0001
        maxValue: 0.5
        scaleType: UNIT_LOG_SCALE
EOF
```

run

```
gcloud beta ml jobs submit training ${JOB_NAME} \
  --package-path=trainer \
  --module-name=trainer.task \
  --staging-bucket="${TRAIN_BUCKET}" \
  --region=us-central1 \
  --config=config.yaml \
  -- \
  --train_data_paths="gs://cloud-ml-data/mnist/train.tfr.gz" \
  --eval_data_paths="gs://cloud-ml-data/mnist/eval.tfr.gz" \
  --output_path="${TRAIN_PATH}/output"
```

