# O-RAN Software Community Non-RT RIC Deployment

## Requirements
vCPU: 4
RAM: 30 GB
SO: Ubuntu 20.04
Kubernetes: 1.19+
Docker: v24.0
Helm: v3.5

## Description
This repository contain all commands to deploy the OSC Non-RT RIC (G release) and a Standalone ONAP (SMO) version running only the SMO functions needed to integrate with the Non-RT RIC.
This file is based on OSC documentation available at https://wiki.o-ran-sc.org/display/REL/G+Release.

OBS: Until this moment the OSC Non-RT RIC G release is under implementation. In this way, we use the repository master branch to clone all files. However, since G Release is done, we need to update this README file.

## Cloning the OSC Non-RT RIC repository
First login as root:
```
sudo su
sudo apt update -y && sudo apt upgrade -y
```
From main branch
```
git clone "https://gerrit.o-ran-sc.org/r/it/dep"
```
From g-release branch (if the main branch contain other release):

```
git clone "https://gerrit.o-ran-sc.org/r/it/dep" -b g-release 
```

Clone the Near-RT RIC repository only to install the requirements:
```
git clone "https://gerrit.o-ran-sc.org/r/ric-plt/ric-dep"
```
Install requirements:
```
cd ric-dep/bin
sudo ./install_common_templates_to_helm.sh
```
OBS: do not install kuberentes and helm using the "*install_k8s_and_helm.sh*" from *ric-dep* repository. This script install a unsupported kubernetes version!

Replace the *dep/bin/prepare-common-templates* file for the file below:
```
#!/bin/bash -x
################################################################################
#   Copyright (c) 2019 AT&T Intellectual Property.                             #
#   Copyright (c) 2019 Nokia.                                                  #
#                                                                              #
#   Licensed under the Apache License, Version 2.0 (the "License");            #
#   you may not use this file except in compliance with the License.           #
#   You may obtain a copy of the License at                                    #
#                                                                              #
#       http://www.apache.org/licenses/LICENSE-2.0                             #
#                                                                              #
#   Unless required by applicable law or agreed to in writing, software        #
#   distributed under the License is distributed on an "AS IS" BASIS,          #
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.   #
#   See the License for the specific language governing permissions and        #
#   limitations under the License.                                             #
################################################################################


ROOT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null && pwd )"

#Check for helm3
IS_HELM3=$(helm version -c --short|grep -e "^v3")

# Start Helm local repo if there isn't one
HELM_REPO_PID=$(ps -x | grep  "helm serve" | grep -v "grep" | awk '{print $1}')

if [ -z "$HELM_REPO_PID" ]
then
    if [ -z $IS_HELM3 ]
    then
      nohup helm serve >& /dev/null &
    else
      nohup helm servecm --port=8879 --context-path=/charts  --storage local --storage-local-rootdir /root/.cache/helm/repository/local/ >& /dev/null &
    fi
fi

# Check if servecm plugin is ready to serve request
command='curl --silent --output /dev/null  http://127.0.0.1:8879/charts'
for i in $(seq 1 5)
do $command && s=0 && break || s=$? && echo "Error connecting chartmuseum server. Retrying after 5s" && sleep 5;
done

if [ $s -gt 0 ]
then
        echo "Cmd to test chartmuseum failed with ($s): $command"
        exit $s
fi

# Package common templates and serve it using Helm local repo
if [ $IS_HELM3 ]
then 
  eval $(helm env |grep HELM_REPOSITORY_CACHE)
  HELM_LOCAL_REPO="${HELM_REPOSITORY_CACHE}/local/"
  mkdir -p $HELM_LOCAL_REPO
else 
  HELM_HOME=$(helm home)
  HELM_LOCAL_REPO="${HELM_HOME}/repository/local/"
fi

COMMON_CHART_VERSION=$(cat $ROOT_DIR/../ric-common/Common-Template/helm/ric-common/Chart.yaml | grep version | awk '{print $2}')
helm package -d /tmp $ROOT_DIR/../ric-common/Common-Template/helm/ric-common
cp /tmp/ric-common-$COMMON_CHART_VERSION.tgz $HELM_LOCAL_REPO

AUX_COMMON_CHART_VERSION=$(cat $ROOT_DIR/../ric-common/Common-Template/helm/aux-common/Chart.yaml | grep version | awk '{print $2}')
helm package -d /tmp $ROOT_DIR/../ric-common/Common-Template/helm/aux-common
cp /tmp/aux-common-$AUX_COMMON_CHART_VERSION.tgz $HELM_LOCAL_REPO

NONRTRIC_COMMON_CHART_VERSION=$(cat $ROOT_DIR/../ric-common/Common-Template/helm/nonrtric-common/Chart.yaml | grep version | awk '{print $2}')
helm package -d /tmp $ROOT_DIR/../ric-common/Common-Template/helm/nonrtric-common
cp /tmp/nonrtric-common-$NONRTRIC_COMMON_CHART_VERSION.tgz $HELM_LOCAL_REPO

helm repo index $HELM_LOCAL_REPO


# Make sure that helm local repo is added
helm repo remove local
helm repo add local http://127.0.0.1:8879/charts

```

Run the new *prepare-common-templates* file:

```
./dep/bin/prepare-common-templates
```
If everithing went well, we can now deploy the Non-RT RIC.

Deploy ONAP (SMO) standalone version and Non-RT RIC:

```
sudo dep/bin/deploy-nonrtric -f dep/nonrtric/RECIPE_EXAMPLE/example_recipe.yaml
```
The expected exit is:
```
To do
```
