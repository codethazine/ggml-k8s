
# Run GGML models on Kubernetes

## Deploy Llama and Mistral using cheap AWS machines with GGML and Kubernetes!

![llamacppbeiac_ratioed_upscaled]()

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://opensource.org/licenses/MIT)

The purpose of this repo is being a proof-of-concept for a scalable GGML/llama.cpp based backend on Kubernetes.

----

### How to deploy
#### 1. Setup
Make sure you have the following installed:
- AWS CLI
- aws-iam-authenticator
- Docker
- kubectl
- eksctl

Then setup your AWS credentials by running the following commands:
```bash
export AWS_PROFILE=your_aws_profile
aws configure --profile your_aws_profile
```
Proceed to change the following files
1. .env:
Create a `.env` file, following the `.env.example` file, with the following variables:
- `AWS_REGION`: The AWS region to deploy the backend to.
- `MIN_CLUSTER_SIZE`: The minimum number of nodes to have on the Kubernetes cluster.
- `EC2_INSTANCE_TYPE`: The EC2 instance type to use for the Kubernetes cluster's node group.
- `ACM_CERTIFICATE_ARN`: The ARN of the ACM certificate to use for the domain.
- `DOMAIN`: The domain to use for the backend.

Currently only Route53 has been tested and is supported for the domain and ACM for the certificate. Make sure to have the Route53 hosted zone created and the ACM certificate validated.

2. models.yaml:
Add your models as shown in the `Uploading new models` section.

#### 2. Deploy
Initialize the Terraform infrastructure by running:
```bash
make deploy-terraform-aws
```
Then initialize the Kubernetes cluster by running:
```bash
make init-cluster-aws
```

#### 3. Enjoy
To test the deployed models with curl:
1. Get the filename from the url, e.g. from https://huggingface.co/TheBloke/Luna-AI-Llama2-Uncensored-GGUF/resolve/main/luna-ai-llama2-uncensored.Q4_K_M.gguf the basename would be `luna-ai-llama2-uncensored.Q4_K_M.gguf`
2. Remove the extension and replace `_` and `.` with `-` and add `.api.$(YOURDOMAIN)` at the end
3. Run requests on the model using the same OAI endpoints and adding the model basename from 1. on the `"model"` section of the data

Example:
```
curl https://luna-ai-llama2-uncensored-q4-k-m.api.example.com/v1/chat/completions \
-H "Content-Type: application/json" \
-d '{
    "model": "luna-ai-llama2-uncensored.Q4_K_M.gguf",
    "messages": [
        {"role": "user", "content": "How are you?"}
    ],
    "stream": true
}'
```
TODO: Create a proxy redirecting requests to the correct services automatically instead of having a different service API url for each model.

### Uploading new models
To upload a new model, identify the model's url, prompt template, requested resources and change the `models.yaml` file by adding the model following this example structure:
```yaml
  - url: "https://huggingface.co/TheBloke/Luna-AI-Llama2-Uncensored-GGUF/resolve/main/luna-ai-llama2-uncensored.Q4_K_M.gguf"
    promptTemplate: |
      You are a helpful AI assistant.
      USER: {{.Input}}
      ASSISTANT:
    resources:
      requests:
        cpu: 8192m
        memory: 16384Mi
```

Then, run the following command:
```bash
make update-kubernetes-cluster
```
This will automatically update the backend with the new model. Make sure to have the necessary resources available on the Kubernetes cluster to run the model.

### Destroying the backend
To destroy the Kubernetes cluster and backend resources run:
```bash
make destroy-terraform-aws
```

----

## Extra considerations
- The backend is currently deployed on a single node, which is not recommended for production environments. Change the `terraform/aws/14-eks-node-group.tf` according to your needs.
- The requests can run parallely thanks to an abstracted thread pool, through the use of multiple LocalAI horizontally scaled server instances (shout-out to the [LocalAI](https://github.com/mudler/LocalAI) team btw).
- When a promptTemplate is defined, this is also used for the `/v1/completions` endpoint. This might be fixed in the future on LocalAI's end, in the meanwhile, if you just need to use the `/v1/completions` endpoint, make sure to not define the promptTemplate for the model on the `models.yaml` file at all.

### TO-DOs:
  - [ ] Add a proxy to redirect requests to the correct service and potentially collect all the /v1/models responses on a single endpoint
  - [ ] Solve thread safety issues on [Llama.cpp](https://github.com/ggerganov/llama.cpp/issues/3960)
  - [ ] Make the backend more scalable by adding more nodes to the Kubernetes cluster automatically through an autoscaling group
  - [ ] Test the backend on GPU enabled nodes
  - [ ] Add support for other cloud providers

Feel free to open an issue or a PR if you have any suggestions or questions!

### Authors 
[danielgross](https://github.com/codethazine) and [codethazine](https://github.com/codethazine).