# DevOps For AI
##### Updated by: Laura Edell, Sr Data Scientist 
##### Last Updated: 1/10/2019

This workshop will help you to understand how to build the CICD pipeline for Data Science work. We will be using the VSTS Team Project build and release tool for it. 

## Prerequisite
- Request Azure ML Python SDK
- Request to get [Azure Subscription Whitelisted](https://github.com/Azure/AzureMLPreview/#prerequisites) for Azure ML Python SDK(not needed once AML SDK is GA)
- VSTS Team Project


## Steps Performed in the Build Pipeline:

1. Prepare the python environment
2. Get or Create the workspace
3. Submit Training job on the remote DSVM / Local Python Env
4. Register model to workspace
5. Create Docker Image for Scoring Webservice
6. Copy and Publish the Artifacts to Release Pipeline

## Steps Performed in the Release Pipeline
In Release pipeline we deploy the image created from the build pipeline to Azure Container Instance and Azure Kubernetes Services

### Deploy on AKS
1. Prepare the python environment
2. Deploy on AKS
    - Create AKS and create a new webservice on AKS with the scoring docker image

        OR

    - Get the existing AKS and update the webservice with new image created in Build Pipeline
3. Test the scoring image

### Deploy on ACI
1. Prepare the python environment
2. Create ACI and Deploy webservice image created in Build Pipeline
3. Test the scoring image

### Repo Details

#### Environment Setup

- requirements.txt : It consist of list of python packages which are needed by the train.py to run successfully on host agent (locally).

- install_requirements.sh : This script prepare the python environment i.e. install the Azure ML SDK and the packages specified in requirements.txt

#### Config Files
All the scripts inside the ./aml_config are config files. These are the files where you need to provide details about the subscription, resource group, workspace, conda dependencies, remote vm, AKS etc.

- config.json : This is a mandatory config file. Provide the subscription id, resource group name and workspace name you want to create. If you have already created the workspace, provide the existing workspace details in here.

- conda_dependencies.yml : This is a mandatory file. This files contains the list of dependencies which are needed by the training/scoring script to run. This file is used to prepare environment for the local run(user managed/system managed) and docker run(local/remote).

- security_config.json : This file contains the credentials to the remove vm where we want to train the model. This config is used by the script 02-AttachTrainingVM.py to attach remote vm as a compute to the workspace. Attaching remote vm to workspace is one time operation. It is recommended not to publish this file with credentials populated in it. You can put the credentials, run the 02-AttachTrainingVM.py manually and clear the credentials before pushing it to git.

- aks_webservice.json : This is an optional config. If you already have an AKS attached to your workspace, then provide the details in this file. If not, you do not have to check in this file to git.

#### Build Pipeline Scripts

The script under ./aml_service are used in build pipeline. All the scripts starting with 0 are the one time run scripts. These are the scripts which need to be run only once. There is no harm of running these scripts every time in build pipeline.

- 00-WorkSpace.py : This is a onetime run script. It reads the workspace details from ./aml_config/config.json file and create (if workspace not available) or get (existing workspace). 

- 01-Project.py : This is a onetime run script. It registers the root directory as project. It is not included as a step in build pipeline.

- 02-AttachTrainingVM.py : This is a onetime run script. It attaches a remote VM to the workspace. It reads the config from ./aml_config/security_config.json. It is not included as a step in build pipeline.

- 10-TrainOnLocal.py : This scripts triggers the run of ./training/train.py script on the local compute(Host agent in case of build pipeline). If you are training on remote vm, you do not need this script in build pipeline. All the training scripts (1x) generates an output file aml_config/run_id.json which records the run_id and run history name of the training run. run_id.json is used by 20-RegisterModel.py to get the trained model.

- 11-TrainOnLocalEnv.py : Its functionality is same as 10-TrainOnLocal.py, the only difference is that it creates a virtual environment on local compute and run training script on virtual env.

- 12-TrainOnVM.py : As we want to train the model on remote VM, this script is included as a task in build pipeline. It submits the training job on remote vm. 

- 20-RegisterModel.py : It gets the run id from training steps output json and registers the model associated with that run along with tags. This scripts outputs a model.json file which contains model name and version. This script included as build task.

- 30-CreateScoringImage.py : This takes the model details from last step, creates a scoring webservice docker image and publish the image to ACR. This script included as build task. It writes the image name and version to image.json file.

#### Deployment/Release Scripts
File under the directory ./deployment/ are used in release pipeline. They are basically to deploy the docker image on AKS and ACI and publish webservice on them. 

- deployOnAci.py : This script reads the image.json which is published as an artifact from build pipeline, create aci cluster and deploy the scoring web service on it. It writes the scoring service details to aci_webservice.json

- deployOnAks.py : This script reads the image.json which is published as an artifact from build pipeline, create aks cluster and deploy the scoring web service on it. If the aks_webservice.json file was checked in with existing aks details, it will update the existing webservice with new Image. It writes the scoring service details to aks_webservice.json

#### Test Scripts
The test scripts in ./test directory/ are part of the release pipeline as well. They test if the webservice deployed on AKS/ACI is working or not.

- AciWebServiceTest.py : Reads the ACI info from aci_webservice.json and test it with sample data.

- AksWebServiceTest.py : Reads the AKS info from aks_webservice.json and test it with sample data.

#### Training/Scoring Scripts

- ./training/train.py : This is the model training code. It uploads the model file to AML Service run id once the training is successful. This script is submitted as run job by all the 1x scripts.

- ./scoring/score.py : This is the score file used to create the webservice docker image. There is a conda_dependencies.yml in this directory which is exactly same as the one in aml_config. These two files are needed by the 30-CreateScoringImage.py scripts to be in same root directory while creating the image.

**Note: In CICD Pipeline, please make sure that the working directory is the root directory of the repo.**  

