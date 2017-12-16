Image classification task with Tensorflow at scale on Google cloud platform along with model deployment and versioning


Step 1
Create a VM instance(lets say Ubuntu 16.04 LTS) on the cloud(a small machine with 1 core and 3.75 GB ram would work)

Step 2
SSH into the machine from the browser

Step3

Install docker on the machine(in our case Ubuntu 16.04 LTS)
You can have a look at the commands(https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04)

1. curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

2. sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

3. sudo apt-get update

4. apt-cache policy docker-ce

5. sudo apt-get install -y docker-ce

6. sudo systemctl status docker


Step 4
Clone the repository

git clone https://github.com/DataScienceDelhi/dsdevent1.git


Step 5
Start a container with tensorflow tag 1.2.0

sudo docker run -it -v `pwd`:/notebooks tensorflow/tensorflow:1.2.0

After you get a running jupyter and a token screen close this window directly

Step 6
List containers

sudo docker ps

sudo docker exec -it <container-id> /bin/bash

This command will take you inside container with root permissions where we will maintain the environment and so that you can package it and take it to any other machine(be it your local machine)

Step 7

apt-get update

apt-get install vim

Step 8

cd dsdevent1

Step 9 
Install Gcloud sdk

export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"

echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

apt-get update

apt-get install google-cloud-sdk

gcloud init

Step 10

Also, we use Apache Beam (running on Cloud Dataflow) and PIL to preprocess the images into embeddings, so make sure to install the required packages:
```
pip install -r requirements.txt
```

Step 11
Create a bucket in your GCS as per the below instructions
Upload dict.txt, train_set.csv, eval_set.csv and preprocessed files to directly jump to Cloud ML Training step
declare USER='puneet'
so bucket name should be flower_${USER}
```
gsutil cp flower1/dict.txt gs://flower_${USER}/
gsutil cp flower1/train_set.csv gs://flower_${USER}/
gsutil cp flower1/eval_set.csv gs://flower_${USER}/
```

Step 12
```
declare PROJECT=$(gcloud config list project --format "value(core.project)")
```
```
declare JOB_ID="flowers_puneet_$(date +%Y%m%d_%H%M%S)"
```
```
declare BUCKET="gs://flower_${USER}"
```
```
declare DICT_FILE="${BUCKET}/dict.txt"
```
```
declare MODEL_NAME=flower
```
```
declare VERSION_NAME=v1

```

Step 13
```
python trainer/preprocess.py --input_dict "$DICT_FILE" --input_path "${BUCKET}/train_set.csv" --output_path "${BUCKET}/preproc/train" --cloud
```

Step 14
```
python trainer/preprocess.py --input_dict "$DICT_FILE" --input_path "${BUCKET}/eval_set.csv" --output_path "${BUCKET}/preproc/eval" --cloud
```

Step 15
Upload preprocessed data to skip above steps
gsutil cp -r flower1/* gs://flower_${USER}/


Step 16
```
# Training on CloudML
gcloud ml-engine jobs submit training "$JOB_ID" --stream-logs --module-name trainer.task --package-path trainer --staging-bucket "${BUCKET}" --region us-central1 --runtime-version=1.2 -- --output_path "${BUCKET}/training" --eval_data_paths "${BUCKET}/preproc/eval*" --train_data_paths "${BUCKET}/preproc/train*"
```

Step 17
```
gcloud ml-engine models create "$MODEL_NAME" --regions us-central1
```
```
gcloud ml-engine versions create "$VERSION_NAME" --model "$MODEL_NAME" --origin "${BUCKET}/training/model" --runtime-version=1.2
```
```
gcloud ml-engine versions set-default "$VERSION_NAME" --model "$MODEL_NAME"
```
```
gsutil cp gs://cloud-ml-data/img/flower_photos/daisy/100080576_f52e8ee070_n.jpg daisy.jpg
```
```
python -c 'import base64, sys, json; img = base64.b64encode(open(sys.argv[1], "rb").read()); print json.dumps({"key":"0", "image_bytes": {"b64": img}})' daisy.jpg &> request.json
```

Step 18
```
gcloud ml-engine predict --model ${MODEL_NAME} --json-instances request.json

=================================