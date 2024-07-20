# 🔀 Query-Router 
Dynamically routes incoming model requests to appropriate LLM based on their varying complexities, thus optimizing response retrieval (by prompting the sufficient model) and saving costs (by not consecutively inferencing larger sized models). Routing can be guided by a [dataset](https://github.com/MuhammadBinUsman03/Query-Router?tab=readme-ov-file#routing-dataset-) for your own use-case and then two routing strategies are provided:
- [Embedding Based Router](https://github.com/MuhammadBinUsman03/Query-Router?tab=readme-ov-file#embedding-based-router-) - Deployed on AWS
- [Classification Based Router](https://github.com/MuhammadBinUsman03/Query-Router?tab=readme-ov-file#classification-based-router-)

The comparisons of both strategies is [here.](https://github.com/MuhammadBinUsman03/Query-Router?tab=readme-ov-file#comparison-)

![queryroute](https://github.com/user-attachments/assets/3e581f3e-2eb9-4fe8-8834-d578127b2f54)

## 📑 Routing Dataset
You can curate your own dataset for your use-case. However, a ~15K [sample dataset](https://huggingface.co/datasets/Muhammad2003/routing-dataset) has been uploaded on 🤗HuggingFace, mapping each query to approrpriate model (3B, 7B, 30B, 70B).
![image](https://github.com/user-attachments/assets/d3269a32-ffe4-4f76-b0d6-8f627544a7d5)

# 🛢️ Embedding Based Router
It is based on a [KNN router by PulzeAI](https://github.com/pulzeai-oss/knn-router/tree/main) which is a Go-Server that generates a ranked target list for a query based on its K-Nearest Neighbors. Additionally, we fine-tune our own [Embedding model](https://huggingface.co/Muhammad2003/router-embedding) based on [BAAI/bge-base-en-v1.5](https://huggingface.co/BAAI/bge-base-en-v1.5/tree/main) on the above mentioned routing dataset. Then after generating deployment artifacts as described below, the routing server is deployed on an AWS-EC2 instance.

![EmbedRouter](https://github.com/user-attachments/assets/0e4b2a72-fa15-428f-9a76-246c9f43cd89)


Setup procedure for the embedding based router is given next.
## Fine-tuning the embedding model
We will fine-tune our own [router-embedding](https://huggingface.co/Muhammad2003/router-embedding) model based on [BAAI/bge-base-en-v1.5](https://huggingface.co/BAAI/bge-base-en-v1.5/tree/main) on the above mentioned routing dataset. The embedding fine-tuning is shown in the [`Embed_FineTune.ipynb`](https://github.com/MuhammadBinUsman03/Query-Router/blob/main/Embed_FineTune.ipynb) where we leverage [SentenceTransformers](https://www.sbert.net/index.html) training our embedding model with loss function `BatchAllTripletLoss`. The training progress is logged on [WandB](https://wandb.ai/home):

![image](https://github.com/user-attachments/assets/b4f61af6-9080-47e0-84a7-c0dc7fb9e1db)

### Points & Targets Dependencies
To generate deployment artifacts, we need following dependencies:
- `points.jsonl`: JSONL-formatted file containing points and their respective categories and embeddings. Each line should contain the following fields: `point_uid`, `category`, and `embedding`.
- `targets.jsonl`: JSONL-formatted file containing the targets and their respective scores for each point. Each line should contain the following fields: `point_uid`, `target`, and `score`.

The [`PointsAndTargets.ipynb`](https://github.com/MuhammadBinUsman03/Query-Router/blob/main/PointsAndTargets.ipynb) can generate these dependencies from our above fine-tuned embedding model and routing dataset, feel free to edit the functions accordignly if your dataset labels are different. After generating push these files to the same embedding model repository on Hub.

## Deployment on AWS-EC2
Navigate to AWS-EC2 Dashboard, launch an EC2 with Ubuntu-OS and 30GB Disk Volume (to avoid disk space shortage/Instance freezing up) and remaining default configurations which are all suitable under the free tier. Let's setup all the dependencies.

Now setup a Python Virtual Environment.
```bash
sudo apt update
sudo apt install python3.12-venv
python3 -m venv .venv
source .venv/bin/activate
```
Let's install and setup Docker on the EC2.
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc
do
  sudo apt-get remove -y "$pkg"
done
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```
Also install Go-lang on the instance.
```bash
sudo apt install golang-go
go version
```
### Creating artifacts
Clone the KNN-router repository.
```bash
git clone https://github.com/pulzeai-oss/knn-router.git
cd knn-router/deploy/pulze-intent-v0.1
```
Install few more dependencies and and authenticate you HF account with `--token`
```bash
pip install transformers huggingface_hub
huggingface-cli login --token ''
```
Install and Initialize Git LFS and clone your embedding model repository from HF which also contains the points and targets dependencies.
```bash
sudo apt-get update && sudo apt-get install git-lfs && git lfs install
git clone https://huggingface.co/Muhammad2003/router-embedding
```
Now you can generate artifacts by providing the `--points-data-path` and `--scores-data-path` from the cloned repository. Then `embeddings.snapshot` and `scores.db` artifacts are generated, which then you can also push to the HF repository back to complete the repository for deployment and delete the the current repo.
```bash
../../scripts/gen-artifacts.sh --points-data-path ./router-embedding/points.jsonl --scores-data-path ./router-embedding/targets.jsonl --output-dir .

huggingface-cli upload Muhammad2003/router-embedding ./embeddings.snapshot embeddings.snapshot
huggingface-cli upload Muhammad2003/router-embedding ./scores.db scores.db
sudo rm -r ./router-embedding
```


### Starting the services
Download the finalized HF repo.
```bash
huggingface-cli download Muhammad2003/router-embedding --local-dir .dist --local-dir-use-symlinks=False
```
Edit the `Docker-compose.yml` file and finally start the server
```bash
sed -i 's|--model-id=/srv/run/embedding-model|--model-id=/srv/run/|' docker-compose.yml
sudo docker compose up -d --build
sudo docker ps -a
```
![image](https://github.com/user-attachments/assets/3c95ab59-390c-4d14-b819-24acdeb9a122)

### Inference
You can get routing output by a CURL request.
```bash
curl -s 127.0.0.1:8888/ \
    -X POST \
    -d '{"query":"How does pineapple on pizza sound?"}' \
    -H 'Content-Type: application/json' | jq .
```
RESPONSE
```bash
  "hits": [
    {
      "id": "801917cd-12de-4dfa-a18a-a8ef51681741",
      "category": "3B",
      "similarity": 0.99916637
    },
    {
      "id": "32def154-7906-4c25-a17a-8536f38b6e43",
      "category": "30B",
      "similarity": 0.9991118
    },
    {
      "id": "b724d01a-3041-40e3-8339-938aada6e9f1",
      "category": "3B",
      "similarity": 0.99910575
    },
    {
      "id": "1a08d6c4-333e-423f-a9ca-0a50fb1115b4",
      "category": "3B",
      "similarity": 0.99910486
    },
    {
      "id": "1657366b-358d-4e7e-8390-50579500fa1c",
      "category": "3B",
      "similarity": 0.9991038
    },
    {
      "id": "d6b85ae6-c82f-4a5f-a294-62b00bf65710",
      "category": "3B",
      "similarity": 0.9990984
    },
    {
      "id": "0052e853-a9f7-4d87-bf7d-deed3a43e23b",
      "category": "3B",
      "similarity": 0.9990958
    },
    {
      "id": "45bb68b0-bd9d-42b0-8510-dac9d615c8e8",
      "category": "3B",
      "similarity": 0.9990936
    },
    {
      "id": "bc87a3ea-c4a5-4587-a0e5-9dff42632c48",
      "category": "3B",
      "similarity": 0.99909335
    },
    {
      "id": "b7af3297-4b3b-4430-8360-e2f95d144727",
      "category": "3B",
      "similarity": 0.9990907
    }
  ],
  "scores": [
    {
      "target": "3B",
      "score": 0.9
    },
    {
      "target": "30B",
      "score": 0.1
    }
  ]
}
```

# Classification Based Router
A simple but ineffective alternative is to train a text classifier on the same data to output the correct label/class for the appropriate model given an input query. [`TinyLlama_RouterClassifier.ipynb`](https://github.com/MuhammadBinUsman03/Query-Router/blob/main/TinyLlama_RouterClassifier.ipynb) guides fine-tuning [TinyLlama/TinyLlama-1.1B-Chat-v0.6](https://huggingface.co/TinyLlama/TinyLlama-1.1B-Chat-v0.6) on the routing dataset for classification task.
![image](https://github.com/user-attachments/assets/0503e1cf-da7e-4041-a8b3-afacd937377a)

# Comparison
Overall classifier may be effecive in outputs but incurs greater inference/storage costs and has high latency, meanwhile embedding router are cosst effective and has faster-response times thus more suitable for large scale systems.
| Aspect                 | Tiny LLaMA Classifier                                      | Embedded KNN Router                                  |
|------------------------|-------------------------------------------------|-----------------------------------------------------|
| Inference Cost         | High (4-5 GB model size)                        | Low (~500 MB model size)                            |
| Resource Requirements  | Significant computational power and storage     | Minimal computational power and storage             |
| GPU Requirement        | Requires GPUs                                   | Does not require GPUs                               |
| Accuracy               | High, capable of handling complex tasks         | Adequate for most routing tasks                     |
| Latency                | High latency, slower response times             | Low latency, faster response times                  |
| Performance            | High accuracy, detailed training                | Almost the same as Tiny LLaMA in practical scenarios |
| Scalability            | Challenging due to high resource demands and costs | Easily scalable, suitable for rapid scaling         |



