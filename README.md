tensorflow on CloudML
====

refs: https://cloud.google.com/ml/docs/quickstarts/training

### training local

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
JOB_NAME=<your job name>
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
# Clear the output from any previous local run.
rm -rf output/
# Train locally.
python -m trainer.task \
  --train_data_paths=gs://cloud-ml-data/mnist/train.tfr.gz \
  --eval_data_paths=gs://cloud-ml-data/mnist/eval.tfr.gz \
  --output_path=output
```

### training CloudML

setup
```
JOB_NAME=<your job name>
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

