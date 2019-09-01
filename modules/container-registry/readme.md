# Container Registry

**Expected Outcome:**

- 100 level introduction to Amazon Elastic Container Registry.

- Containers pushed to ECR.

**Lab Requirements:** Cloud9 IDE

**Average Lab Time:** 10-15 minutes

## Introduction

In this module, we’re going to take our already containerized
applications and push them to [Amazon Elastic Container Registry
(ECR)](https://aws.amazon.com/ecr/). Amazon ECR is a fully-managed
Docker container registry that makes it easy for developers to store,
manage, and deploy Docker container images. Amazon ECR is integrated
with Amazon Elastic Container Service (ECS), simplifying your
development to production workflow.

### Deploy ECR Repositories

We will first deploy a CloudFormation template which configures our ECR
Repositories.

CloudFormation creates the following resources:

- `PetStorePostgreSQLRepository` - ECR Repository for PostgresSQL
    containers.

- `PetStoreFrontendRepository` - ECR Repository for the front-end
    containers.

#### Step 1

Using the Cloud9 ‘terminal\`, change to this modules’ directory by
running:

    cd ~/environment/aws-modernization-workshop/modules/container-registry

#### Step 2

We need to customize the ECR repository names so as to distinguish them
from those used by the other Workshop attendees. To do this, run the
following command. This command will replace the &lt;UserName&gt;
placeholder in the CloudFormation template with your unique user name.

    sed -i "s/<UserName>/${USER_NAME}/" ~/environment/aws-modernization-workshop/modules/container-registry/petstore-ecr-cf-resources.yaml

#### Step 3

Now deploy the CloudFormation template using the `aws cli` tool.

    aws cloudformation create-stack --stack-name "${USER_NAME}-petstore-ecr" \
    --template-body file://petstore-ecr-cf-resources.yaml \
    --capabilities CAPABILITY_NAMED_IAM

#### Step 4

Wait for the Template to finish deploying by running the following
command:

    until [[ `aws cloudformation describe-stacks --stack-name \
    "${USER_NAME}-petstore-ecr" --query "Stacks[0].[StackStatus]" \
    --output text` == "CREATE_COMPLETE" ]]; \
    do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";   \
    sleep 30; done && echo "The Stack is built at `date` - Please proceed"

#### Pushing the Petstore images to Amazon Elastic Container Registry (ECR)

Before we can deploy our Petstore application to an orchestrator, we
need to push our Petstore images to **ECR**.

#### Step 1

Log into your Amazon ECR registry using the helper provided by the
AWS CLI.

    eval $(aws ecr get-login --no-include-email)

> **Note**
>
> Ignore any *WARNNING* messages.

#### Step 2

Use the AWS CLI to get information about the two Amazon ECR repositories
that were created for you from the CloudFormation template. One
repository will be for the Petstore PostgreSQL backend and the other
will be for the Petstore web frontend.

    aws ecr describe-repositories --repository-name \
    ${USER_NAME}-petstore_postgres ${USER_NAME}-petstore_frontend

Example output:

    {
        "repositories": [
            {
                "registryId": "123456789012",
                "repositoryName": "petstore_postgres",
                "repositoryArn": "arn:aws:ecr:us-west-2:123456789012:repository/<User Name>-petstore_postgres",
                "createdAt": 1533757748.0,
                "repositoryUri": "123456789012.dkr.ecr.us-west-2.amazonaws.com/<User NAme>-petstore_postgres"
            },
            {
                "registryId": "123456789012",
                "repositoryName": "petstore_frontend",
                "repositoryArn": "arn:aws:ecr:us-west-2:123456789012:repository/<User Name>-petstore_frontend",
                "createdAt": 1533757751.0,
                "repositoryUri": "123456789012.dkr.ecr.us-west-2.amazonaws.com/<User Name>-petstore_frontend"
            }
        ]
    }

#### Step 3

Verify that your Docker images exist by running the docker images
command.

    docker images

Example Output:

    REPOSITORY                          TAG                 IMAGE ID            CREATED             SIZE
    containerize-application_petstore   latest              3190952ddfc8        39 minutes ago      799MB
    postgres                            9.6                 8d9572468d97        4 days ago          230MB
    lambci/lambda                       python3.6           420212d009b3        4 weeks ago         1.03GB
    lambci/lambda                       python2.7           7a436931435e        4 weeks ago         901MB
    lambci/lambda                       nodejs4.3           c0914066d9a8        4 weeks ago         931MB
    lambci/lambda                       nodejs6.10          74b405a65ed4        4 weeks ago         946MB
    lambci/lambda                       nodejs8.10          edf1f613772c        4 weeks ago         960MB
    jboss/wildfly                       11.0.0.Final        a12aa93a45f7        3 months ago        700MB
    maven                               3.5-jdk-7           5f03adaf2bbf        7 months ago        483MB

#### Step 4

Tag the local docker images with the locations of the remote ECR
repositories we created using the CloudFormation template.

    docker tag postgres:9.6 $(aws ecr describe-repositories --repository-name ${USER_NAME}-petstore_postgres \
    --query=repositories[0].repositoryUri --output=text):latest

    docker tag containerize-application_petstore:latest $(aws ecr describe-repositories \
    --repository-name ${USER_NAME}-petstore_frontend \
    --query=repositories[0].repositoryUri --output=text):latest

#### Step 5

Once the images have been tagged, push them to the remote repository.

    docker push $(aws ecr describe-repositories --repository-name ${USER_NAME}-petstore_postgres \
    --query=repositories[0].repositoryUri --output=text):latest

    docker push $(aws ecr describe-repositories --repository-name ${USER_NAME}-petstore_frontend \
    --query=repositories[0].repositoryUri --output=text):latest

You should see the Docker images being pushed with an output similar to
this:

    The push refers to repository [123456789012.dkr.ecr.us-west-2.amazonaws.com/petstore_postgres]
    7856d1f55b98: Pushed
    a125032aca95: Pushed
    fcfc309521a9: Pushed
    4c4e9f97ac56: Pushed
    109402c6a817: Pushed
    6663c6c0d308: Pushed
    ed4da41a79a9: Layer already exists
    7c050956ab95: Layer already exists
    c6fcee3b341c: Layer already exists
    998e6abcfae7: Layer already exists
    df9515382700: Layer already exists
    0fae9a7d0574: Layer already exists
    add4404d0b51: Layer already exists
    cdb3f9544e4c: Layer already exists
    latest: digest: sha256:ca39b6107978303706aac0f53120879afcd0d4b040ead7f19e8581b81c19ecea size: 3243

With the images pushed to Amazon ECR we are ready to deploy them to our
orchestrator. The next modules will show you how to leverage both
[Amazon Elastic Container Service (Amazon
ECS)](http://aws.amazon.com/ecs/), [AWS
Fargate](http://aws.amazon.com/fargate/) and [Amazon
EKS](https://aws.amazon.com/eks/) to orchestrate our containers into
production.

![Choice](../../images/choose.png)
