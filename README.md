# Mocroservice-app-AzureDevops


project for a microservice application deployment using Azure DevOps. It includes the steps needed to dynamically update the Docker image tag in the Kubernetes `deployment.yml` file and deploy it to Azure Kubernetes Service (AKS).

### Project Structure:

```
/microservice-app
|--- Dockerfile
|--- main.py
|--- requirements.txt
|--- azure-pipelines.yml
|--- /k8s
      |--- deployment.yml
```

### Step 1: Microservice Application Code

**main.py (Python Flask Microservice):**

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello from the Microservice!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**Dockerfile:**

```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install the required packages
RUN pip install --no-cache-dir -r requirements.txt

# Expose port 5000 for the Flask application
EXPOSE 5000

# Define environment variable
ENV NAME World

# Run the application
CMD ["python", "main.py"]
```

**requirements.txt:**

```text
Flask==2.0.1
```

### Step 2: Kubernetes Deployment Configuration (`deployment.yml`)

**k8s/deployment.yml (Kubernetes Deployment Configuration with Placeholder for Image Tag):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservice-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: microservice
  template:
    metadata:
      labels:
        app: microservice
    spec:
      containers:
      - name: microservice
        image: <your-acr-name>.azurecr.io/microservice-app:{{imageTag}}  # Placeholder for image tag
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: microservice-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: microservice
```

### Step 3: Azure DevOps Pipeline (`azure-pipelines.yml`)

**azure-pipelines.yml (Pipeline Definition to Build, Push, and Deploy the Docker Image):**

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureSubscription: '<your-azure-subscription-name>'  # Update with your Azure subscription name
  resourceGroup: 'myResourceGroup'                      # Update with your Azure resource group
  aksCluster: 'myAKSCluster'                            # Update with your AKS cluster name
  acrName: '<your-acr-name>'                            # Update with your Azure Container Registry (ACR) name
  imageName: 'microservice-app'

steps:
# Step 1: Get AKS Credentials
- task: AzureCLI@2
  inputs:
    azureSubscription: $(azureSubscription)
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az aks get-credentials --resource-group $(resourceGroup) --name $(aksCluster)

# Step 2: Build and Push the Docker Image
- task: Docker@2
  inputs:
    containerRegistry: '$(acrName).azurecr.io'
    repository: '$(imageName)'
    command: 'buildAndPush'
    dockerfile: '**/Dockerfile'
    tags: |
      $(Build.BuildId)

# Step 3: Replace Image Tag in deployment.yml with Build ID
- script: |
    sed -i 's/{{imageTag}}/$(Build.BuildId)/g' k8s/deployment.yml
  displayName: 'Replace image tag in deployment.yml'

# Step 4: Deploy to AKS using the Updated deployment.yml
- task: Kubernetes@1
  inputs:
    kubernetesServiceConnection: 'YourKubernetesServiceConnection'
    namespace: default
    command: apply
    useConfigurationFile: true
    configuration: 'k8s/deployment.yml'
    secretType: 'dockerRegistry'
    containerRegistryType: 'Azure Container Registry'
    azureSubscriptionEndpoint: '$(azureSubscription)'
    azureResourceManagerConnection: '$(azureSubscription)'
```

### Step 4: Kubernetes Deployment File (`deployment.yml`)

This file contains the placeholder `{{imageTag}}`, which will be dynamically replaced with the actual Docker image tag (e.g., the build ID) in the Azure DevOps pipeline.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservice-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: microservice
  template:
    metadata:
      labels:
        app: microservice
    spec:
      containers:
      - name: microservice
        image: <your-acr-name>.azurecr.io/microservice-app:{{imageTag}}  # This will be replaced by Build ID
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: microservice-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: microservice
```

### Explanation of the Pipeline Flow:

1. **Get AKS Credentials**: The pipeline fetches the AKS credentials to ensure that it can deploy the microservice to your Kubernetes cluster.
   
2. **Build and Push Docker Image**: Azure DevOps builds a Docker image from the Dockerfile and pushes it to your Azure Container Registry (ACR). The build ID is used as the image tag.

3. **Replace Image Tag in `deployment.yml`**: The `sed` command replaces the `{{imageTag}}` placeholder in `deployment.yml` with the actual build ID (the Docker image tag). This ensures that Kubernetes pulls the correct version of the Docker image.

4. **Deploy to AKS**: Finally, the pipeline deploys the updated `deployment.yml` to the AKS cluster using the Kubernetes task. The service is exposed using a LoadBalancer, making the application accessible.

### Step 5: Creating Service Connections in Azure DevOps

1. **Azure Subscription Service Connection**: In Azure DevOps, go to the project settings, and add a new service connection for your Azure subscription. This connection allows the pipeline to authenticate and deploy resources on Azure.

2. **Azure Container Registry (ACR) Service Connection**: Ensure you have a connection to your ACR, allowing Docker images to be pushed from the pipeline.

3. **Kubernetes Service Connection**: Create a Kubernetes service connection to your AKS cluster so that the pipeline can deploy the Docker container using `kubectl`.

---

This is the complete setup for a simple CI/CD pipeline in Azure DevOps for deploying a microservice to AKS. The Docker image is built and pushed to ACR, and the Kubernetes deployment is updated automatically with the correct image tag.
