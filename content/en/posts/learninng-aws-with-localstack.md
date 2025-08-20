+++
title = "Lerning AWS with LocalStack"
tags = ["terraform", "aws", "cloud", "devops", "localstack", "fastapi", "python"]
date = "2025-06-24"
lang = "en"
+++

### Introduction

I have worked with various cloud providers, such as GCP, IBM and Azure. Yes, they are some of the most used, but I lack experience with the most popular one: AWS. Since I've used it superficially, I wanted to practice with some of the available 
services, although I no longer have any free credits left in my account... I began researching until I came across LocalStack, a project that emulates various AWS services in a container. So, that's exactly what I'll cover in this post: how
I set up a small environment using different AWS services through LocalStack, Terraform and Python.


- [Objective](#objective)
- [LocalStack installation](#localstack-installation)
- [Tags](#tags)
- [DynamoDB](#dynamodb)
- [Terraform](#terraform)
- [Lambda and S3](#lambda-and-s3)
- [DynamoDB Streams, Secret Manager y SES](#dynamodb-streams-secret-manager-y-ses)

### Objective

Since I needed to use multiple services, I developed a FastApi service that connects to DynamoDB to create, list and delete Product entities. Then, a Lambda function was enabled to send an email using SES, triggered by DynamoDB Streams. Exists 
[a list](https://docs.localstack.cloud/references/coverage/) with all mocked services.

{{< images/centered-image src="/posts/learninng-aws-with-localstack/intro.png" >}}

The code is available in a GitHub [public repository](https://github.com/MartiMarch/fake-aws.git).

### LocalStack installation

LocalStack can be deployed directly with Docker or, as I choose, on Kubernetes via the official [helm chart](https://github.com/localstack/helm-charts/tree/main/charts/localstack). The AWS CLI is also required, configured to point to LocalStack IP (
192.168.2.83 in this case).

```bash
sudo snap install aws-cli --classic
# I created alias "awsl" to be faster -> alias awsl='aws --endpoint-url=http://192.168.2.83:4566'

awsl configure
# AWS Access Key ID [None]: fake
# AWS Secret Access Key [None]: fake
# Default region name [None]: fake
# Default output format [None]: json
```

### Tags

AWS lets you tag the resources. Both 'Groups' and 'Resource Groups Tagging API' can be used for that. But LocalStack doesn't mock any of them. Nevertheless, even if LocalStack doesn't implement the command to filter resources by group, I'll tag
every resource to follow best practices.  

In this case, the project called 'fake-aws' is deployed in PRE and PRO environments, so I've established the following naming convention: {{< inlines/black-inline "fake-aws-pre" >}} and {{< inlines/black-inline "fake-aws-pro" >}}.

```bash
awsl resource-groups create-group \
  --name "fwk-aws-pre" \
  --display-name "fwk-aws-pre" \
  --cli-input-json file://resource-pre-query.json

awsl resource-groups list-group-resources --group fwk-aws-pre
```

Tags must be set during resource creation. Groups can also be used as filters to organize resources as needed. For example, you could use a group to filter by {{< inlines/black-inline "fwk-aws" >}} the project and the 
environment {{< inlines/black-inline "pre" >}}.

```bash
awsl resource-groups create-group \
  --name "fwk-aws-pre" \
  --display-name "fwk-aws-pre" \
  --cli-input-json file://resource-pre-query.json

awsl resource-groups list-groups
```

```json
{
  "Name": "fwk-aws-pre",
  "Description": "Recursos del entorno pre para el proyecto fwk-fake",
  "ResourceQuery": {
    "Type": "TAG_FILTERS_1_0",
    "Query": "{\"ResourceTypeFilters\":[\"AWS::AllSupported\"],\"TagFilters\":[{\"Key\":\"project\",\"Values\":[\"fwk-fake\"]},{\"Key\":\"env\",\"Values\":[\"pre\"]}]}"
  }
}
```

This flag tags every resource with the previously established convention tags:

```bash
--tags '[
    {Key=project,Value=fwk-fake},
    {Key=env,Value=pre}
]'
```

### DynamoDB

Before DynamoDB deployment, it's need to create in KMS the encrypted communication credentials.

```bash
awsl kms create-key \
  --description "Clave SSE para el DynamoDB de pre" \
  --tags TagKey=project,TagValue=fwk-aws-pre TagKey=env,TagValue=pre \
  --key-usage ENCRYPT_DECRYPT \
  --customer-master-key-spec SYMMETRIC_DEFAULT

awsl kms list-keys
```

```json
{
  "Keys": [
    {
      "KeyId": "0f291778-4cb1-43a2-bc5f-e47638b4854c",
      "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/0f291778-4cb1-43a2-bc5f-e47638b4854c"
    }
  ]
}
```

I have also created an alias to avoid using the ID secret.

```bash
awsl kms create-alias \
  --alias-name alias/fwk-aws-pre-dynamodb-sse \
  --target-key-id 0f291778-4cb1-43a2-bc5f-e47638b4854c

awsl kms tag-resource \
  --key-id alias/fwk-aws-pre-dynamodb-sse \
  --tags TagKey=project,TagValue=fwk-aws-pre TagKey=env,TagValue=pre
  
awsl kms list-aliases
```

```json
{
  "Aliases": [
    {
      "AliasName": "alias/fwk-aws-pre-dynamodb-sse",
      "AliasArn": "arn:aws:kms:us-east-1:000000000000:alias/fwk-aws-pre-dynamodb-sse",
      "TargetKeyId": "0f291778-4cb1-43a2-bc5f-e47638b4854c",
      "CreationDate": "2025-06-14T21:53:18.071846+02:00"
    }
  ]
}
```

Once alias is created, DynamoDB can be deployed:

```bash
awsl dynamodb create-table \
    # The table name only accepts alphanumeric characters, underscores, and must be between 3 and 255 characters long
    --table-name fake-aws-pre-dynamodb \
    # List of required attributes
    --attribute-definitions \
        AttributeName=productId,AttributeType=S \
        AttributeName=company,AttributeType=S \
        AttributeName=provider,AttributeType=S \
        AttributeName=price,AttributeType=N \
        AttributeName=stock,AttributeType=N \
    --key-schema \
        # If it is a HASH, it acts as a unique key — unless a RANGE is used on another attribute. In that case, duplicates are allowed as long as the RANGE attribute has a different value
        # One attribute must be of type HASH, no exceptions.
        AttributeName=productId,KeyType=HASH \
    # You can create Global Secondary Indexes (GSI) to query your data using attributes other than the primary key
    --global-secondary-indexes \
        '[
            {
                "IndexName": "IndexCompany",
                "KeySchema": [
                    {"AttributeName": "company", "KeyType": "HASH"}
                ],
                "Projection": {
                    "ProjectionType": "ALL" # Incluye todos los atributos del ítem en los resultados
                },
                "ProvisionedThroughput": { # solo se usa si tu tabla está en modo provisionado
                    "ReadCapacityUnits": 10,
                    "WriteCapacityUnits": 5
                }
            },
            {
                "IndexName": "IndexProveder",
                "KeySchema": [
                    {"AttributeName": "provider", "KeyType": "HASH"}
                ],
                "Projection": {
                    "ProjectionType": "ALL"
                },
                "ProvisionedThroughput": {
                    "ReadCapacityUnits": 10,
                    "WriteCapacityUnits": 5
                }
        }]' \
    # PROVISIONED → Preconfigured capacity, PAY_PER_REQUEST → On-demand billing (pay per use)
    --billing-mode PROVISIONED \ 
    --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=5 \
    # Enable DynamoDB Streams
    --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
    --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/fwk-aws-pre-dynamodb-sse \
    # The key is stored in the KMS service and is used to encrypt communications
    --tags Key=project,Value=fwk-aws-pre Key=env,Value=pre
```

DynamoDB tables can be listed to check if they were created correctly.

```bash
awsl dynamodb list-tables
```

```json
{
    "TableNames": [
        "fake-aws-pre-dynamodb"
    ]
}
```

With that, we get a DynamoDB working database (NoSQL), although without a service connected to it, it isn't very useful. So, I've created a FastApi Python project with Poetry to create, delete and list product entities. It implements a basic hexagonal
pattern to quickly develop the service and focus on services integration, leaving aside backend code. Even if it's not perfect, it will be helpful to integrate AWS services.

```bash
poetry init

# This command will guide you through creating your pyproject.toml config.
#
# Package name [fake-aws]:  fwk-aws-micro
# Version [0.1.0]:  0.0.0
# Description []:
# Author [marti <marti_march@protonmail.com>, n to skip]:  n
# License []:
# Compatible Python versions [^3.12]:  ^3.9
#
# Would you like to define your main dependencies interactively? (yes/no) [yes] no
# Would you like to define your development dependencies interactively? (yes/no) [yes] no
# Generated file
#
# [tool.poetry]
# name = "fwk-aws-micro"
# version = "0.0.0"
# description = ""
# authors = ["Your Name <you@example.com>"]
# readme = "README.md"
#
# [tool.poetry.dependencies]
# python = "^3.9"
#
#
# [build-system]
# requires = ["poetry-core"]
# build-backend = "poetry.core.masonry.api"
#
#
# Do you confirm generation? (yes/no) [yes] yes

poetry add boto3
poetry add pydantic
poetry add fastapi
```

Python project follows this folder structure:

```
├── adapters
│   ├── dynamodb.py
│   ├── __init__.py
│   └── ses.py
├── api.py
└── models
    ├── __init__.py
    ├── ProductDTO.py
    ├── ProductMapper.py
    └── Product.py
```

Adapter class for DynamoDB used to create, eliminate, and list Product entity.

```python
from fakeaws.models.ProductMapper import ProductMapper
from fakeaws.models.ProductDTO import ProductDTO
from fakeaws.models.Product import Product
from decimal import Decimal
from typing import List
from uuid import UUID
import boto3


class DynamoDB:
    ENDPOINT = 'http://192.168.2.83:4566'

    def __init__(self):
        self._service = boto3.resource(
            'dynamodb',
            region_name='fake',
            aws_access_key_id='fake',
            aws_secret_access_key='fake',
            endpoint_url=self.ENDPOINT,
        )

    @property
    def service(self):
        return self._service

    def delete(self, product_id: str):
        table = self.service.Table("fake-aws-pre-dynamodb")
        table.delete_item(
            Key={
                "productId": product_id
            },
            ReturnValues="ALL_OLD"
        )

        return {
            "status": "ok",
            "message": f"Product with id {product_id} deleted"
        }

    def save(self, product_dto: ProductDTO) -> ProductDTO:
        product = ProductMapper.dto_to_product(product_dto)
        products_table = self.service.Table('fake-aws-pre-dynamodb')
        products_table.put_item(
            Item={
                "productId": str(product.id),
                "company": product.company,
                "provider": product.provider,
                "price": Decimal(str(product.price)),
                "stock": product.stock,
            }
        )

        return ProductMapper.product_to_dto(product)

    def list(self) -> List[ProductDTO]:
        products_dto: list[ProductDTO] = []

        products_table = self.service.Table('fake-aws-pre-dynamodb')
        response = products_table.scan()
        items = response.get('Items', [])
        for item in items:
            product = Product(
                company=item.get('company'),
                provider=item.get('provider'),
                price=item.get('price'),
                stock=item.get('stock'),
            )
            product.id = UUID(item.get('productId'))
            product_dto: ProductDTO = ProductMapper.product_to_dto(product)
            products_dto.append(product_dto)

        return products_dto
```

Regarding the models packages, I created an entity for Product, a DTO for the adapters and a mapper to map the DTO object to the entity and vice versa.

```python
import uuid


class Product:

    def __init__(self, company: str, provider: str, price: float, stock: int):
        self._id = uuid.uuid4()
        self._company = company
        self._provider = provider
        self._price = price
        self._stock = stock

    @property
    def id(self) -> uuid.UUID:
        return self._id

    @id.setter
    def id(self, id: uuid.UUID):
        self._id = id

    @property
    def company(self) -> str:
        return self._company

    @company.setter
    def company(self, company: str):
        self._company = company

    # There're a more getters/setters...
```

```python
from pydantic import BaseModel


class ProductDTO(BaseModel):
    productId: str = None
    company: str
    provider: str
    price: float
    stock: int
```

```python
from fakeaws.models.ProductDTO import ProductDTO
from fakeaws.models.Product import Product


class ProductMapper:

    @classmethod
    def product_to_dto(cls, product: Product) -> ProductDTO:
        return ProductDTO(
            productId=str(product.id),
            company=product.company,
            provider=product.provider,
            price=product.price,
            stock=product.stock
        )

    @classmethod
    def dto_to_product(cls, dto: ProductDTO) -> Product:
        product = Product(
            company=dto.company,
            provider=dto.provider,
            price=dto.price,
            stock=dto.stock
        )
        if dto.productId is not None:
            product.productId = dto.productId

        return product
```

Once the microservice is deployed, requests can be sent to the API, storing the information into DynamoDB. 

- Product creation

```bash
curl \
  --location 'http://localhost:8080/dynamodb' \
  --header 'Content-Type: application/json' \
  --data '{
      "company": "bic",
      "provider": "alpine",
      "price": 9.99,
      "stock": 10
    }'
```

```json
{
    "productId": "d6012d61-f807-4b85-ad11-337ad414a556",
    "company": "bic",
    "provider": "alpine",
    "price": 9.99,
    "stock": 10
}
```

- Listing products

```bash
curl --location 'http://localhost:8080/dynamodb'
```

```json
[
    {
        "productId": "c375b6ab-562a-4d20-a5cc-710286b67be6",
        "company": "bic",
        "provider": "alpine",
        "price": 9.99,
        "stock": 10
    },
    {
        "productId": "d6012d61-f807-4b85-ad11-337ad414a556",
        "company": "bic",
        "provider": "alpine",
        "price": 9.99,
        "stock": 10
    }
]
```

- Deletion of a Product

```bash
curl \
  --location  \
  --request DELETE 'http://localhost:8080/dynamodb/d6012d61-f807-4b85-ad11-337ad414a556'
```

```json
{
    "status": "ok",
    "message": "Product with id d6012d61-f807-4b85-ad11-337ad414a556 deleted"
}
```

### Terraform

I'm gonna add more functionalities to the microservice while LocalStack services mocks are deployed. However, infrastructure orchestration can become a problem if it's managed using CLI only, especially as the number of AWS services and configuration
increase. So, to make it easier, I've created a Terraform project to define everything as IaC.


Folders are being structured to differentiate the PRE and PRO infrastructure environments, with a module for each AWS service. There are base variables to set the project name and LocalStack endpoint every time the modules are invoked, as well as
environment variables for the PRE and PRO environments.

```
├── base
│   ├── base.tfvars
│   └── lambda.py
├── modules
│   ├── dynamodb
│   │   └── main.tf
│   ├── group
│   │   └── main.tf
│   ├── lambda
│   │   └── main.tf
│   ├── s3
│   │   ├── lambda.zip
│   │   └── main.tf
│   └── secret_manager
│       └── main.tf
├── pre
│   ├── pre.tf
│   └──pre.tfvars
└── pro
    ├── pro.tf
    └── pro.tfvars
```

To transcribe what has been previously deployed to Terraform, the DynamoDB and group modules have been created.

```terraform
# Groups module
# terraform/modules/group/main.tf

variable "project" {
    type = string
}

variable "env" {
    type = string
}

resource "aws_resourcegroups_group" "this" {
  name         = "${var.project}-${var.env}"
  description  = "Grupo de recursos para ${var.project} en el entorno ${var.env}"

  resource_query {
    query = jsonencode({
      ResourceTypeFilters = ["AWS::AllSupported"]
      TagFilters = [
        {
          Key = "project"
          Values = [var.project]
        },
        {
          Key = "env"
          Values = [var.env]
        }
      ]
    })
    type = "TAG_FILTERS_1_0"
  }

  tags = {
    project = var.project
    env     = var.env
  }
}
```

```terraform
# Dynamodb module
# terraform/modules/dynamodb/main.tf

variable "project" {
    type = string
}

variable "env" {
    type = string
}

resource "aws_kms_key" "this" {
    description              = "Clave SSE para el DynamoDB de pre"
    key_usage                = "ENCRYPT_DECRYPT"
    customer_master_key_spec = "SYMMETRIC_DEFAULT"
    enable_key_rotation      = true

    tags = {
        project = "${var.project}"
        env     = "${var.env}"
    }
}

resource "aws_kms_alias" "this" {
  name          = "alias/${var.project}-${var.env}-dynamodb-see"
  target_key_id = aws_kms_key.this.key_id
}

resource "aws_dynamodb_table" "this" {
    name           = "${var.project}-${var.env}-dynamodb"
    billing_mode   = "PROVISIONED"
    read_capacity  = 10
    write_capacity = 5
    hash_key       = "productId"

    attribute {
        name = "productId"
        type = "S"
    }
    attribute {
        name = "company"
        type = "S"
    }
    attribute {
        name = "provider"
        type = "S"
    }

    global_secondary_index {
        name            = "IndexCompany"
        hash_key        = "company"
        projection_type = "ALL"
        read_capacity   = 10
        write_capacity  = 5
    }

    global_secondary_index {
        name            = "IndexProveder"
        hash_key        = "provider"
        projection_type = "ALL"
        read_capacity   = 10
        write_capacity  = 5
    }

    stream_enabled   = true
    stream_view_type = "NEW_AND_OLD_IMAGES"
    server_side_encryption {
        enabled     = true
        kms_key_arn = "arn:aws:kms:us-east-1:000000000000:alias/${var.project}-${var.env}-dynamodb-sse"
    }

    tags = {
        project = "${var.project}"
        env = "${var.env}"
    }
}

output "table_stream_arn" {
    value = aws_dynamodb_table.this.stream_arn
}
```

*.tfvars have also been created for variables common to all environments along with variables specific to each environment.

```terraform
# terraform/pre/pre.tfvars
env = "pre"
```

```terraform
# terraform/base/base.tfvars
project = "fake-aws"
localstack_endpoint = "http://192.168.2.83:4566"
```

```bash
terraform plan -var-file="../base/base.tfvars" -var-file="pre.tfvars"
terraform apply -var-file="../base/base.tfvars" -var-file="pre.tfvars" -auto-approve
```

### Lambda and S3

As I mentioned in the introduction, I want to use Lambda to send an email. It's a Python function with a certain name; Localstack will only print the logs to the pod when invoked. However, an S3 bucket also needs to be created since Lambda has to
download the source code from somewhere, so two modules have been created, one for S3 and one for Lambda.

```terraform
# S3 module
# terraform/modules/s3

variable "project" {
    type = string
}

variable "env" {
    type = string
}

# Bucket definition
resource "aws_s3_bucket" "this" {
    bucket = "${var.project}-${var.env}-s3"

    tags = {
        project = "${var.project}"
        env     = "${var.env}"
    }
}

# Needed permissions to use the binary file pushed into the bucket by lambda
resource "aws_s3_bucket_ownership_controls" "this" {
    bucket = aws_s3_bucket.this.id

    rule {
        object_ownership = "BucketOwnerPreferred"
    }
}

resource "aws_s3_bucket_public_access_block" "this" {
    bucket = aws_s3_bucket.this.id

    block_public_acls       = true
    block_public_policy     = true
    ignore_public_acls      = true
    restrict_public_buckets = true
}

resource "aws_s3_object" "this" {
    bucket = aws_s3_bucket.this.id
    key    = "lambda.zip"
    source = "${path.module}/lambda.zip"
    etag   = filemd5("${path.module}/lambda.zip")

    tags = {
        project = "${var.project}"
        env     = "${var.env}"
    }
}

# All these outputs are necessary for the creation of the lambda
output "bucket_name" {
  value = aws_s3_bucket.this.bucket
}

output "object_key" {
  value = aws_s3_object.this.key
}

output "object_etag" {
  value = aws_s3_object.this.etag
}
```

```terraform
# Lambda module
# terraform/modules/lambda

variable "project" {
    type = string
}

variable "env" {
    type = string
}

variable "bucket_name" {
    type = string
}

variable "object_key" {
    type = string
}

variable "object_etag" {
    type = string
}

variable "table_stream_arn" {
    type = string
}

# Defining the IAM role required to access the bucket
resource "aws_iam_role" "this" {
    name = "${var.project}-${var.env}-lambda-role"
    assume_role_policy = jsonencode({
    Version = "2012-10-17"
        Statement = [
            {
                Action    = "sts:AssumeRole"
                Effect    = "Allow"
                Principal = {
                    Service = "lambda.amazonaws.com"
                }
            }
        ]
    })

    tags = {
        project = "${var.project}"
        env     = "${var.env}"
    }
}

# Lamda function definition
resource "aws_lambda_function" "this" {
    s3_bucket        = var.bucket_name
    s3_key           = var.object_key
    # Function name, if it does not match the script name it will fail
    function_name    = "${var.project}-${var.env}-lambda-function"
    role             = aws_iam_role.this.arn
    handler          = "lambda.handler"
    source_code_hash = base64sha256(var.object_etag)
    runtime          = "python3.11"
    timeout          = 60
    memory_size      = 512
    tags = {
        project = "${var.project}"
        env     = "${var.env}"
    }
}

resource "aws_iam_role_policy_attachment" "this" {
    role       = aws_iam_role.this.name
    policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

Any lambda function can be called using the AWS CLI. That's what I did, but I only saw an error log in the Localstack pod.

```bash
 Error: waiting for Lambda Function (fake-aws-pre-lambda-function) create: unexpected state 'Failed', wanted target 'Active'. last error: InternalError: Error while creating lambda: Docker not available

   with module.fake-aws-pre-lambda.aws_lambda_function.this,
   on ../modules/lambda/main.tf line 73, in resource "aws_lambda_function" "this":
   73: resource "aws_lambda_function" "this" {
```

For this reason, I decided to create a very basic lambda that prints a "Hello world!" and try out different modifications.


```python
def handler(event, context):
    print("Hello world!")
```

```bash
zip lambda.zip labda.py

awsl lambda create-function \
  --function-name test \
  --runtime python3.11 \
  --handler lambda.handler \
  --role arn:aws:iam::000000000000:role/lambda-role \
  --zip-file fileb://lambda.zip

awsl lambda invoke \
  --function-name test \
  --payload '{"test":"value"}' \
  --cli-binary-format raw-in-base64-out \
  output.txt
```

After much research, I determined that the inability to execute the lambda stems from two causes:
- The Localstack pod cannot execute lambda functions because the container is unable to orchestrate containers
- The lambda function container is unable to return the response to the Localstack pod, which points to the localhost domain

To resolve the issues, I modified the Helm chart, adding certain environment variables, creating a new container that inherits from the one already configured but using the Docker CLI, and mounting the Docker socket volume.

```Dockerfile
FROM localstack/localstack:latest

RUN apt -y update \
&& apt -y install \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg \
     software-properties-common \
&& install -m 0755 -d /etc/apt/keyrings \
&& curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
&& chmod a+r /etc/apt/keyrings/docker.asc \
&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "${VERSION_CODENAME}") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null \
&& apt -y update \
&& apt install -y \
     docker-ce \
     docker-ce-cli \
     containerd.io \
     docker-buildx-plugin \
     docker-compose-plugin
```

```yaml
image:
  repository: "pre.docker.nexus.com/localstack:0.0.0"

volumes:
  - name: docker-socket
    hostPath:
      path: /var/run/docker.sock

volumeMounts:
  - name: docker-socket
    mountPath: /var/run/docker.sock

extraEnvVars:
  - name: LOCALSTACK_HOST
    value: "192.168.2.83:4566"
  - name: DEBUG
    value: "1"
  - name: DOCKER_BRIDGE_IP
    value: "192.168.2.83"
  - name: GATEWAY_LISTEN
    value: "0.0.0.0:4566"
```

With all of that enabled, it is now possible to invoke the lambda function without errors.

```bash
awsl lambda invoke --function-name test --payload '{"test":"value"}' --cli-binary-format raw-in-base64-out output.txt
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

### DynamoDB Streams, Secret Manager y SES

The last piece to implement is sending the email from the lambda when an event occurs in DynamoDB. In this case, I implemented sending via both SES and SMTP. Because I want to use SMTP, it's necessary to use Secret Manager to manage the password 
for the email that sends the email, injecting the value with an environment variable when running Terraform, and retrieving the secret from the lambda function using the boto3 library.

```terraform
# Secret maanger module
# terraform/modules/secret_manager

variable "project" {
  type = string
}

variable "env" {
  type = string
}

# Variable que se inyectan como varaible de entorno
variable "sender_email" {
  type      = string
  sensitive = true
}

variable "sender_email_secret" {
  type      = string
  sensitive = true
}

variable "receiver_email" {
  type      = string
  sensitive = true
}


resource "aws_secretsmanager_secret" "this" {
  name = "${var.project}-${var.env}-email-secret"

  tags = {
    project = "${var.project}"
    env     = "${var.env}"
  }
}

resource "aws_secretsmanager_secret_version" "this" {
  secret_id     = aws_secretsmanager_secret.this.id
  secret_string = jsonencode({

    sender_email        = var.sender_email
    sender_email_secret = var.sender_email_secret
    reciever_email      = var.receiver_email
  })
}

output "email_secret_arn" {
    value = aws_secretsmanager_secret.this.arn
}
```

The environment variables that Terraform will use to configure the secrets value.

```bash
export TF_VAR_sender_email="fake_sender_email@gmail.com"
export TF_VAR_sender_email_secret="fake_password"
export TF_VAR_receiver_email="fake_receiver_email@gmail.com"
```

To enable DynamoDB streams, you need to modify the assigned permissions, both to be able to enable DynamoDB streams and to manage the secret.

```terraform
# Lambda module
# terraform/modules/lambda

variable "email_secret_arn" {
    type = string
    sensitive = true
}


resource "aws_iam_role_policy" "this" {
    name = "${var.project}-${var.env}-lambda-dynamodb-policy"
    role = aws_iam_role.this.id

    policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
        {
            Effect = "Allow"
            # Permisos para DynamoDB Streams
            Action = [
                "dynamodb:DescribeStream",
                "dynamodb:GetRecords",
                "dynamodb:GetShardIterator",
                "dynamodb:ListStreams",
            ]
            # ARN del stream de DynamoDB
            Resource = var.table_stream_arn
        },
        {
            # Permission to read a secret in Secrets Manager
            Effect = "Allow"
            Action = [
                "secretsmanager:GetSecretValue"
            ]
            Resource = var.email_secret_arn
        }
    ]
    })
}

# Bind a DynamoDB Stream to the Lambda so that it runs automatically when there are changes to the table
resource "aws_lambda_event_source_mapping" "this" {
    event_source_arn = var.table_stream_arn
    function_name    = aws_lambda_function.this.arn
    starting_position = "LATEST"
    enabled = true
}
```

As for the Python code for the lambda, it consists of two functions, each implementing either SES or SMTP.

```python
from email.mime.text import MIMEText
import smtplib
import boto3
import json
import os


GMAIL_SMTP_DOMAIN: str = 'smtp.gmail.com'
GMAIL_SMTP_PORT: int = 587
REGION: str = 'us-east-1'
LOCALSTACK_ENDPOINT = 'http://192.168.2.83:4566'


def handler(event, context):
    # SMTP
    # send_email_via_smtp()

    # SES
    send_email_via_ses()


def send_email_via_smtp():
    sm_client = boto3.client('secretsmanager')
    secret_content = json.loads(
        sm_client.get_secret_value(SecretId=os.environ['SM_SECRET_NAME'])
    )
    sender_email = secret_content['sender_email']
    sender_email_secret = secret_content['sender_email_secret']
    receiver_email = secret_content['receiver_email']

    msg = MIMEText('Hello World!')
    msg['Subject'] =  f'Product has been removed'
    msg['From'] = sender_email_secret
    msg['To'] = receiver_email

    with smtplib.SMTP(GMAIL_SMTP_DOMAIN, GMAIL_SMTP_PORT) as gmail:
        gmail.starttls()
        gmail.login(sender_email, sender_email_secret)
        gmail.sendmail(sender_email, receiver_email, msg.as_string())


def send_email_via_ses():
    ses = boto3.client(
        'ses',
        region_name='us-east-1',
        endpoint_url=LOCALSTACK_ENDPOINT,
        aws_access_key_id='fake-key',
        aws_secret_access_key='fake-key'
    )
    sm_client = boto3.client('secretsmanager')
    secret_content = json.loads(
        sm_client.get_secret_value(SecretId=os.environ['SM_SECRET_NAME'])
    )
    sender_email = secret_content['sender_email']
    receiver_email = secret_content['receiver_email']

    ses.send_email(
        Source=sender_email,
        Destination={
            'ToAddresses': [receiver_email]
        },
        Message={
            'Subject': {
                'Data': f'Product has been removed'
            },
            'Body': {
                'Text': {
                    'Data': 'Hello World!'
                }
            }
        }
    )
```

With all this, the list of services I had planned would be implemented. Perhaps in the future I'll expand the project by adding a CI pipeline and a Helm chart with ArgoCD. For now, I'll leave it here.
