Image classification task with Tensorflow at scale on Google cloud platform along with model deployment and versioning


Step 1
Create an VM instance(lets say Ubuntu 16.04 LTS) on the cloud(a small machine with 1 core and 3.75 GB ram would work)

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




Also, we use Apache Beam (running on Cloud Dataflow) and PIL to preprocess the images into embeddings, so make sure to install the required packages:
```
pip install -r requirements.txt
```

Then, you may follow the instructions

Create flowers1 bucket in your GCS
Upload dict.txt, train_set.csv, eval_set.csv
```
declare PROJECT=$(gcloud config list project --format "value(core.project)")
```
```
declare JOB_ID="flowers_puneet_$(date +%Y%m%d_%H%M%S)"
```
```
declare BUCKET="gs://flowers1"
```
```
declare DICT_FILE="${BUCKET}/dict.txt"
```
```
declare MODEL_NAME=flowers
```
```
declare VERSION_NAME=v1

```
```
python trainer/preprocess.py --input_dict "$DICT_FILE" --input_path "${BUCKET}/train_set.csv" --output_path "${BUCKET}/preproc/train" --cloud
```
```
python trainer/preprocess.py --input_dict "$DICT_FILE" --input_path "${BUCKET}/eval_set.csv" --output_path "${BUCKET}/preproc/eval" --cloud
```
```

# Training on CloudML
gcloud ml-engine jobs submit training "$JOB_ID" --stream-logs --module-name trainer.task --package-path trainer --staging-bucket "$BUCKET" --region us-central1 --runtime-version=1.2 -- --output_path "${BUCKET}/training" --eval_data_paths "${BUCKET}/preproc/eval*" --train_data_paths "${BUCKET}/preproc/train*"
```
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
```
gcloud ml-engine predict --model ${MODEL_NAME} --json-instances request.json
