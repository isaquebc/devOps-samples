# devOps-samples

In this folder I have a example of how to deploy a new instance to each PR, sync this instance on each PR update and terminate this instance when the PR is closed or converted to draft.

I can't share the official code, this is a pocket version of a task that a did. Just thinking that is usefull, so, I want to share


### Problem

I'm working in a company that have alot of updates in a day, it's not a huge company, but sometimes we have alot of PR's to test, and we didn't have a good way to it. We start it creating a experimental environment, where we was able to run e2e tests and access the environment to check this version, but when we had 2 or 3 things to test at same time, 1 or 2 tasks have to wait the tests of first task.

### Solution

With this update, we just create a PR, and it will deploy a new instance. 

We had already a ami of our experimental and development environments. Using one of these ami, I started to launch a instance when a PR got the status "Ready to Review", and deploy the PR's branch on this instance, creating a registry on route 53 and generating a certificate using certbot, doing everything using what we already had, without change alot our workflow and architecture.


### Requirements

To do it, I just assume that you have some knowledge of aws devOps, github and docker, you can use this as example and change everything that you need to fit it in your project

To start, you may already have a web application in production, in my case, I have a frontend builded in ReactJs using vitejs, and a backend using django, our infra was on aws.

Before code, you should have some setup to do:

- Create a role, the role on aws save your instance permisions, it will be linked with your instance. save this information, you will use it (profile role name)  (https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html).
- Create a ami, of your current instance (https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/tkv-create-ami-from-instance.html)
- Choose a instance type to craete this instance (https://aws.amazon.com/ec2/instance-types/)
- Have the key.pem name that is conected with your ami, to ssh access
- Know the security Group and subnet id that you want to connect (with wrong security group, you may have problems to conect with instance, using ssh, http or any port)
- Choose a instance name pattern, I just use "PR-[PR-NUMEBR]"
- Choose a instance description, I just use "https://github.com/[GIT-ACCOUNT]/[GIT-REPO]/pull/[PR-NUMEBR]" to make a link between the instance and PR
- Create a user to connect with  github, you can save the "AWS_SECRET_ACCESS_KEY" and "AWS_ACCESS_KEY_ID" on your github secrets, and give the permissions that you want or need.
- Have a recordset setted up on route53 to save the domain of new environment
- The files of github actions should be on path ".github/workflows/" on your project folder


#### Labels

I use labels to identify the PR's instance status, so, if you want have the same labels, you should create the labels before run the action. The labels are:

##### New Env

- "Building..."
- "Starting new instance..."
- "Registering on route 53..."
- "Deploying into new instance..."
- "ON-LINE"
- "ERROR to set new environment"

##### Sync Env

- "Building..."
- "Updating environment..."
- "ON-LINE"
- "ERROR to update environment"

##### Kill Env

- "Terminating enviroment..."
- "Cleaning route 53..."
- "OFF"

### Steps

#### Warging

I'm using docker to build my images, if you are not using docker, you can just skip this step and build the app on the instance using ssh script

#### 1 - Creating Building step

First of all, you should have already a build workflow setted up, I'm using to store my docker images the aws ecr, so, the file that build my frontend is [here](https://github.com/isaquebc/devOps-samples/blob/main/github-actions/env-per-pr/.github/workflows/build-frontend.yml) and my backend build is [here](https://github.com/isaquebc/devOps-samples/blob/main/github-actions/env-per-pr/.github/workflows/build-backend.yml)

#### 2 - Geting ready new Environment 

Here I start the file indicating the workflow name:

```
name: New Env

on:
  pull_request:
    branches: ["development"]
    types: [ready_for_review]
  workflow_dispatch:

```

The on is what will trigger this action exp:

  - ```pull_request``` - Configure this action to run on PR's
    - ```branches``` - It means that will run on PR's to "development" branch
    - ```types``` - What is the action type, in this case when the PR's status change to "ready_for_review"

  - ```workflow_dispatch``` - It means that this workflow call other workflow.

Before start the instance, lets use labels to define the PR's instance status, this will be our first job of this workflow

```
name: New Env

on:
  pull_request:
    branches: ["development"]
    types: [ready_for_review]
  workflow_dispatch:

jobs:
  label-pr:
    runs-on: ubuntu-latest
    steps:
      - name: Remove All Labels
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PR_NUMBER=${{ github.event.pull_request.number }}
          REPO=${{ github.repository }}
          curl -X DELETE \
            -H "Authorization: token $GITHUB_TOKEN" \
            -H "Accept: application/vnd.github.v3+json" \
            https://api.github.com/repos/$REPO/issues/$PR_NUMBER/labels

      - name: Add "Building..." label via GitHub API
        run: |
          curl -s -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github.v3+json" \
            https://api.github.com/repos/${{ github.repository }}/issues/${{ github.event.pull_request.number }}/labels \
            -d '{"labels":["Building..."]}'

```

Inside the job key, we can declare our jobs, we called this label-pr, it will run in a ubuntu-latest machine,
the first step we just use to clean all labels of this PR, and then, adding the label "Building..." all through curl requests, using variables that github give to us already.

The next job will be the build of frontend and backend, all they are independents, "label-pr", "build-frontend", "build-backend" can run simultaneously.

```
  build-frontend:
    uses: ./.github/workflows/build-frontend.yml
    secrets:
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    with:
      IMAGE_TAG: "${{ github.event.pull_request.number }}"

  build-backend:
    uses: ./.github/workflows/build-backend.yml
    secrets:
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    with:
      IMAGE_TAG: "${{ github.event.pull_request.number }}"
```

As you see, we are passing the secrets to the build job, because a workflow that is called by other, don't have access to the parents secrets. I'm using the number of PR to declare the image tag of frontend and backend

The file is like this right now:

```
name: New Env

on:
  pull_request:
    branches: ["development"]
    types: [ready_for_review]
  workflow_dispatch:


jobs:
  # Label PR
  label-pr:
    runs-on: ubuntu-latest
    steps:
      - name: Remove All Labels
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PR_NUMBER=${{ github.event.pull_request.number }}
          REPO=${{ github.repository }}
          curl -X DELETE \
            -H "Authorization: token $GITHUB_TOKEN" \
            -H "Accept: application/vnd.github.v3+json" \
            https://api.github.com/repos/$REPO/issues/$PR_NUMBER/labels

      - name: Add "Building..." label via GitHub API
        run: |
          curl -s -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github.v3+json" \
            https://api.github.com/repos/${{ github.repository }}/issues/${{ github.event.pull_request.number }}/labels \
            -d '{"labels":["Building..."]}'

  # Frontend build
  build-frontend:
    uses: ./.github/workflows/build-frontend.yml
    secrets:
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }} # configured on github repo
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }} # configured on github repo
    with:
      IMAGE_TAG: "${{ github.event.pull_request.number }}"

  # Backend build
  build-backend:
    uses: ./.github/workflows/build-backend.yml
    secrets:
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}  # configured on github repo
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}  # configured on github repo
    with:
      IMAGE_TAG: "${{ github.event.pull_request.number }}"
```




#### 3 - Starting new Environment

Now, we will start a new instance on aws ec2.
We will start adding this job in the end of the file:

```
cd-deploy:
    runs-on: ubuntu-latest
    needs: [build-frontend, build-backend]
    steps:
```

The cd-deploy will run in ubuntu-latest too, but it depends of the success of "build-frontend" and "build-backend" jobs, if it got error on build, this step will be skiped.

```
  - name: Checkout repository
    uses: actions/checkout@v3

  - name: Setup AWS Credentials
    uses: aws-actions/configure-aws-credentials@v2
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }} # configured on github repo
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }} # configured on github repo
      aws-region: [REGION] # You can use the github variables/secrets to get the aws region from the repository or just hardcode it
```

The step "Checkout repository" will update the current code
The step "Setup AWS Credentials" will configure the aws credentials to this job.


```
 - name: Remove All Labels
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      PR_NUMBER=${{ github.event.pull_request.number }}
      REPO=${{ github.repository }}
      curl -X DELETE \
        -H "Authorization: token $GITHUB_TOKEN" \
        -H "Accept: application/vnd.github.v3+json" \
        https://api.github.com/repos/$REPO/issues/$PR_NUMBER/labels

  - name: Add "Starting new instance..." label via GitHub API
    run: |
      curl -s -X POST \
        -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
        -H "Accept: application/vnd.github.v3+json" \
        https://api.github.com/repos/${{ github.repository }}/issues/${{ github.event.pull_request.number }}/labels \
        -d '{"labels":["Starting new instance..."]}'

```

Now, I'm updating the label again, removing all, and adding a label indicating the next step.

```
- name: Start ec2 instance
  id: start_ec2
  run: |
    # Defining environment variables
    AMI_ID=[AMI-ID]
    INSTANCE_TYPE=[INSTANCE-TYPE] 
    KEY_NAME=[GENERATED-KEY-NAME]
    SECURITY_GROUP_ID=[SECURITY-GROUP-ID]
    SUBNET_ID=[SUBNET-ID]
    IAM_INSTANCE_PROFILE=[IAM-INSTANCE-PROFILE-ROLE]
    INSTANCE_NAME="PR-${{ github.event.pull_request.number }}"
    INSTANCE_DESCRIPTION="https://github.com/[GIT-ACCOUNT]/[GIT-REPO]/pull/${{ github.event.pull_request.number }}"

    echo "* * * Starting EC2 instance with name $INSTANCE_NAME and description $INSTANCE_DESCRIPTION"

    # Starting the EC2 instance and get the instance ID
    INSTANCE_ID=$(aws ec2 run-instances \
      --image-id $AMI_ID \
      --count 1 \
      --instance-type $INSTANCE_TYPE \
      --key-name $KEY_NAME \
      --security-group-ids $SECURITY_GROUP_ID \
      --subnet-id $SUBNET_ID \
      --associate-public-ip-address \
      --iam-instance-profile Name=$IAM_INSTANCE_PROFILE \
      --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$INSTANCE_NAME},{Key=Description,Value=$INSTANCE_DESCRIPTION}]" \
      --query 'Instances[0].InstanceId' \
      --output text)
    echo "INSTANCE_ID=$INSTANCE_ID" >> $GITHUB_ENV

    echo "* * * Waiting 'running' status on instance"
    # Wait for the instance status to be running
    aws ec2 wait instance-running --instance-ids $INSTANCE_ID

    
    echo "* * * Geting the public IP of the instance"
    
    # Get the public IP of the instance
    PUBLIC_IP=$(aws ec2 describe-instances \
      --instance-ids $INSTANCE_ID \
      --query 'Reservations[0].Instances[0].PublicIpAddress' \
      --output text)

    # Save variables to be used in the next steps
    echo "PUBLIC_IP=$PUBLIC_IP" >> $GITHUB_ENV
    echo "AMI_ID=$AMI_ID" >> $GITHUB_ENV
    echo "KEY_NAME=$KEY_NAME" >> $GITHUB_ENV
    echo "INSTANCE_NAME=$INSTANCE_NAME" >> $GITHUB_ENV
    DOMAIN="pr${{ github.event.pull_request.number }}.YOUR.DOMAIN"
```
These variables will can declare on github repository secrets, or hardcode it here.

AMI_ID, INSTANCE_TYPE, KEY_NAME, SECURITY_GROUP_ID, SUBNET_ID and IAM_INSTANCE_PROFILE

Its important to save the public ip, and instance id to reuse it in the future.

After start the instance, we just wait it be runing, get the public ip and sabe some variables to reuse.

```
  - name: Remove All Labels
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      PR_NUMBER=${{ github.event.pull_request.number }}
      REPO=${{ github.repository }}
      curl -X DELETE \
        -H "Authorization: token $GITHUB_TOKEN" \
        -H "Accept: application/vnd.github.v3+json" \
        https://api.github.com/repos/$REPO/issues/$PR_NUMBER/labels

  - name: Add "Registering on route 53..." label via GitHub API
    run: |
      curl -s -X POST \
        -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
        -H "Accept: application/vnd.github.v3+json" \
        https://api.github.com/repos/${{ github.repository }}/issues/${{ github.event.pull_request.number }}/labels \
        -d '{"labels":["Registering on route 53..."]}'

```

Here we just clear the labels and add the "Registering on route 53...".

```
  - name: Setup DNS Record on Route 53
    run: |
      # Get the domain from the file
      DOMAIN=$(cat domain_file.txt)
      
      echo "DOMAIN = $DOMAIN" 
      
      # Define the domain for the PR
      HOSTED_ZONE_ID=[HOSTED-ZONE-ID] # You can use the github variables/secrets to get the hosted zone id from the repository or just hardcode it

      echo "* * * Creating the JSON file for DNS entries"

      # Create the JSON file for DNS entries
      echo '{"Comment": "Create A records for '$DOMAIN' and *.'$DOMAIN'","Changes": [{"Action": "UPSERT","ResourceRecordSet": {"Name": "'$DOMAIN'","Type": "A","TTL": 300,"ResourceRecords": [{  "Value": "'$PUBLIC_IP'"}]}},{"Action": "UPSERT","ResourceRecordSet": {"Name": "*.'$DOMAIN'","Type": "A","TTL": 300,"ResourceRecords": [{  "Value": "'$PUBLIC_IP'"}]}}]}' > dns-record.json
      
      # Read the JSON file just to log it
      cat dns-record.json

      echo "* * * Updating the DNS record in Route 53"

      # Update the DNS record in Route 53 using created files
      aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://dns-record.json

```

Here we create the registry domain on our hosted zone, using aws cli and a created json. It's important to have the public ip and domain saved to reuse


```
  - name: Remove All Labels
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      PR_NUMBER=${{ github.event.pull_request.number }}
      REPO=${{ github.repository }}
      curl -X DELETE \
        -H "Authorization: token $GITHUB_TOKEN" \
        -H "Accept: application/vnd.github.v3+json" \
        https://api.github.com/repos/$REPO/issues/$PR_NUMBER/labels

  - name: Add "Deploying into new instance..." label via GitHub API
    run: |
      curl -s -X POST \
        -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
        -H "Accept: application/vnd.github.v3+json" \
        https://api.github.com/repos/${{ github.repository }}/issues/${{ github.event.pull_request.number }}/labels \
        -d '{"labels":["Deploying into new instance..."]}'
```

Removing labels and add the label "Deploying into new instance..."


```
  - name: Access instance via SSH
    run: |
      DOMAIN=pr${{ github.event.pull_request.number }}.YOUR.DOMAIN 
      echo "${{ secrets.AWS_PEM_FILE_CONTENT }}" > key.pem 
      chmod 400 key.pem
```

Here we create a pem file to access instance through ssh.

```
  - name: Access instance via SSH
    run: |
      DOMAIN=pr${{ github.event.pull_request.number }}.YOUR.DOMAIN 
      echo "${{ secrets.AWS_PEM_FILE_CONTENT }}" > key.pem 
      chmod 400 key.pem

      echo "Wait for the instance to be ready using AWS CLI"
      INSTANCE_STATUS=""
      
      # Wait for the instance to be ready
      while [ "$INSTANCE_STATUS" != "running" ]; do
        INSTANCE_STATUS=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query "Reservations[0].Instances[0].State.Name" --output text)
        echo "Waiting for the instance to be ready... Current status: $INSTANCE_STATUS"
        sleep 10
      done
```

Here I just wait the instance be running, when I was implementing, sometimes the instance was not running yet.

```
    printf "* * * Accessing the instance throgh SSH"

    # Runing script throgh SSH
    # Here you need to replace YOUR.DOMAIN with your domain
    ssh -o StrictHostKeyChecking=no -i "key.pem" ubuntu@pr${{ github.event.pull_request.number }}.YOUR.DOMAIN << 'ENDSSH' 
```

After check the instance we start the ssh script
key.pem is the name of the file that we create on the last step, ubuntu is the user of the machine, and the "pr${{ github.event.pull_request.number }}" will be pr42 if your pr is the pr number 42. 

You should replace "YOUR.DOMAIN" to the domain of your choice


```
  ssh -o StrictHostKeyChecking=no -i "key.pem" ubuntu@pr${{ github.event.pull_request.number }}.YOUR.DOMAIN << 'ENDSSH' 
            
    cd /home/ubuntu/[YOUR-PROJECT-FOLDER]

    DOMAIN=pr${{ github.event.pull_request.number }}.YOUR.DOMAIN
    BRANCH_NAME=${{ github.head_ref }}

    git fetch origin $BRANCH_NAME
    git checkout $BRANCH_NAME
    git reset --hard origin/$BRANCH_NAME
```

Here I'm just going to the projecy folder and access current branch to reset to the last changes.

```
  sudo hostnamectl set-hostname PR-${{ github.event.pull_request.number }}

```

This line will change the name of the machine, to easyer indentify where you are.


```
  DEFAULT_DOMAIN="$DOMAIN"
  echo "WEBSITE_URL='https://$DEFAULT_DOMAIN'" >> /home/ubuntu/[YOUR-PROJECT-FOLDER]/backend/backend/settings_local.py

```

Here I just set a variable to django.

```
  WILDCARD_DOMAIN="*.$DOMAIN"
  sudo certbot certonly --standalone -d $DOMAIN -d $WILDCARD_DOMAIN --non-interactive --agree-tos -m [YOUR-EMAIL]

```

Now I'm generating a new certificate using certbot, to the domain, and the wildcard domain.


```
  export AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }} AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }} && aws ecr get-login-password --region [REGION] | docker login --username AWS --password-stdin [ACCOUNT].dkr.ecr.[REGION].amazonaws.com/[REPOSITORY]
```

Logging in on ecr


```
  export DOCKER_REPO=$DOCKER_REPO REF_NAME=$REF_NAME DOCKER_IMAGE_TAG=$DOCKER_IMAGE_TAG DEFAULT_DOMAIN=$DEFAULT_DOMAIN && make deploy_populate

```

Here I'm just runing a make command to deploy the new version using docker compose.

entire [new-env-exp.yml](https://github.com/isaquebc/devOps-samples/blob/main/github-actions/env-per-pr/.github/workflows/new-env-exp.yml) file

