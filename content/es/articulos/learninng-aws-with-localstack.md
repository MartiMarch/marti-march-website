+++
title = "Aprendiendo AWS con localstack"
tags = ["terraform", "aws", "cloud", "devops", "localstack", "fastapi", "python"]
date = "2025-06-24"
lang = "es"
+++

### Introducción

He trabajado con varios proveedores cloud, como por ejemplo GCP, IBM o Azure. Sí, son algunos de los más usados, pero falta el más popular, AWS. Como lo he utilizado muy superficialmente quería practicar sobre algunos de los servicios disponibles, aunque
no me quedan más créditos gratis en la cuenta... Me puse a indagar un poco hasta que encontré LocalStack, un proyecto que simula parcialmente los servicios de AWS en un contenedor. Así que eso es justo lo que voy a comentar en este artículo, como he
creado un pequeño escenario en el que voy usando diferentes servicios de AWS a través de LocalStack, Terraform y Python.

- [Tags](#tags)
- [DynamoDB](#dynamodb)
- [Terraform](#terraform)
- [Lambda y S3](#lambda-y-s3)
- [DynamoDB Streams, Secret Manager y SES](#dynamodb-streams-secret-manager-y-ses)

### Objetivo

Como quiero usar varios servicios, he desarrollado un servicio en FastApi que se conecta a DynamoDB para crear, listar y eliminar la entidad Producto. Después,  he habilitado una función Lambda que envía un correo usando SES y cuyo trigger es 
DynamoDB Streams. Hay [una lista](https://docs.localstack.cloud/references/coverage/) con los servicios mockeados.

{{< images/centered-image src="/posts/learninng-aws-with-localstack/intro.png" >}}

Todo el código generado está almacenado en un [repo público](https://github.com/MartiMarch/fake-aws.git).

### Instalación de LocalStack

Localstack se puede desplegar directamente sobre Docker o, como he escogido yo, sobre Kubernetes con el [chart de Helm](https://github.com/localstack/helm-charts/tree/main/charts/localstack) que tiene publicado. También hay que instalar el CLI de aws
y reapuntarlo a la IP de LocalStack.

```bash
sudo snap install aws-cli --classic
# He creado el alias "awsl" -> alias awsl='aws --endpoint-url=http://192.168.2.83:4566'

awsl configure
# AWS Access Key ID [None]: fake
# AWS Secret Access Key [None]: fake
# Default region name [None]: fake
# Default output format [None]: json
```

### Tags

En AWS se pueden tagear los recursos, para ello se puede emplear los Groups y los Resource Groups Tagging API. Aunque no se puede ejecutar el comando de Localstack que filtra por grupo los recursos, yo iré tageando las cosas para seguir las 
buenas praxis.

En mi caso el proyecto se llama fake-aws, va a estar desplegado en los entornos de PRE y en PRO y, voy a establecer una nomenclatura como la siguiente: {{< inlines/black-inline "fake-aws-pre" >}} y {{< inlines/black-inline "fake-aws-pro" >}}.

```bash
awsl resource-groups create-group \
  --name "fwk-aws-pre" \
  --display-name "fwk-aws-pre" \
  --cli-input-json file://resource-pre-query.json

awsl resource-groups list-group-resources --group fwk-aws-pre
```

Lo que se hace en AWS es poner un tag al recurso en el momento de la creación. Se pueden emplear también los grupos, que no son nada más que filtros para sacar los servicios en base a los tags deseados. Por ejemplo, se puede crear un grupo que filtre
a partir del proyecto {{< inlines/black-inline "fwk-aws" >}} con el entorno {{< inlines/black-inline "pre" >}}.

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

Para asociar cada recurso hay que usar el siguiente tag en el momento de su creación:
```bash
--tags '[
    {Key=project,Value=fwk-fake},
    {Key=env,Value=pre}
]'
```

### DynamoDB

Antes de desplegar DynamoDB es necesario crear en KMS las credenciales necesarias para la encriptación de las comunicaciones.

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

También hay que crear un alias para no arrastrar el ID del secreto usado en la encriptación en la creación de DyanmoDB.

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

Con el alias creado ya se puede pasar a la creación de DynamoDB:

```bash
awsl dynamodb create-table \
    # Nombre de la tabla, solo carácteres alfanuméricos, guiones bajos y de 3 a 255 carácteres
    --table-name fake-aws-pre-dynamodb \
    # Lista de atributos obligatorios
    --attribute-definitions \
        AttributeName=productId,AttributeType=S \
        AttributeName=company,AttributeType=S \
        AttributeName=provider,AttributeType=S \
        AttributeName=price,AttributeType=N \
        AttributeName=stock,AttributeType=N \
    --key-schema \
        # Si es hash es como una clave única a no ser que se utilice RANGE en otro parámetro, entonces sí que se puede duplicar siempre y cuando se tenga el parmámetro marcado con RANGE con un valor diferente. 
        # Sí o sí un parámetro ha de ser de tipo HASH
        AttributeName=productId,KeyType=HASH \
    # Crear índices secundarios globales (GSI) que permiten consultar tus datos usando otros atributos distintos a la clave primaria
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
    # PROVISIONED -> Capacidad configurada, PAY_PER_REQUEST -> Pago por uso
    --billing-mode PROVISIONED \ 
    --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=5 \
    # Habilita DynamoBD streams
    --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
    --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/fwk-aws-pre-dynamodb-sse \
    # Clave almazenada en el servicio KMS, es la clave que cifra las comunicaciones
    --tags Key=project,Value=fwk-aws-pre Key=env,Value=pre
```

Se puede listar las tablas de DynamoDB para revisar si se ha creado correctamente.

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

Y con esto tendríamos la base de datos NoSQL funcionando aunque, sin un servicio que se conecte, no tiene mucha utilidad. He creado un proyecto en Python (Poetry) con FastApi para crear, eliminar y listar la entidad producto. He seguido una hexagonal
básica para desarrollar rápidamente y centrarme en los servicios, dejando de lado el backend. Aunque no va a estar perfecto, va a servir para probar la integración.

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

La estructura del proyecto es la siguiente:

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
    └─── Product.py
```

Clase adaptador de DynamoDB con las operaciones para crear, eliminar y listar la entidad Product.

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

En cuanto al package de los modelos, he creado una entidad para Product, un DTO para los adaptadores y un mapper para pasar de DTO a entidad y viceversa.

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

    # Muchos más getters y setters...
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

Y con todo esto desplegado ya puede lanzarse peticiones al servicio, guardando la información en DynamoDB.

- Creación de un producto

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

- Listar los productos

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

- Eliminación del producto

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

Voy a ir agregando más funcionalidades al micro, además de crear otros servicios con Localstack. El problema es que orquestar la infraestructura desde el CLI de AWS directamente puede llegar a ser engorroso cuando hay que crear varios entornos, por eso,
hay que crear un proyecto de Terraform y definir todo como IAC.

Los directorios se han estructurado para diferenciar la infraestructura en los entornos de PRE y PRO, con un módulo por servicio de AWS. Hay unas variables base, para mantener el mismo nombre de proyecto y el endpoint de Localstack al invocar a los módulos,
además de las variables de entorno de PRE y PRO.

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

Para trascribir lo que se ha desplegado previamente a Terraform, se ha creado los módulos de DynamoDB y group.

```terraform
# Módulo de groups
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
# Módulo de dynamodb
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

También sea han creado los *.tfvars para las variables comunes a todo los entornos junto a las variables específicas de cada entorno.

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

### Lambda y S3

Como comentaba en la introducción, quiero utilizar lambda para enviar un correo. Se trata de una función en Python con cierto nombre, Localstack tan solo imprimirá los logs en el pod cuando se invoque. Aunque también se necesita crear un bucket de S3 ya que
el código fuente ha de ser descargado por lambda desde algún lugar, así que se ha creado dos módulos, uno para S3 y otro para lambda.

```terraform
# Módulo de S3
# terraform/modules/s3

variable "project" {
    type = string
}

variable "env" {
    type = string
}

# Es la definición del bucket
resource "aws_s3_bucket" "this" {
    bucket = "${var.project}-${var.env}-s3"

    tags = {
        project = "${var.project}"
        env     = "${var.env}"
    }
}

# Permisos necesarios para que lambda pueda utilizar el binario subido al bucket
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

# Comprimido donde está la función python
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

# Todos estos outputs son necesarios para la creación de la lambda
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
# Módulo de lambda
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

# Definición del rol de IAM necesario para acceder al bucket
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

# Definición de la función lambda
resource "aws_lambda_function" "this" {
    s3_bucket        = var.bucket_name
    s3_key           = var.object_key
    # Nombre de la función, si no coincide con la del script fallará
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

Cualquier función lambda se puede invocar desde el CLI de AWS. Eso es lo que hice pero, no para de ver un log en el pod de Localstack.

```bash
 Error: waiting for Lambda Function (fake-aws-pre-lambda-function) create: unexpected state 'Failed', wanted target 'Active'. last error: InternalError: Error while creating lambda: Docker not available

   with module.fake-aws-pre-lambda.aws_lambda_function.this,
   on ../modules/lambda/main.tf line 73, in resource "aws_lambda_function" "this":
   73: resource "aws_lambda_function" "this" {
```

Por eso mismo, decidí crear una lambda muy básica que imprime un "Hola mundo!" e ir probando diferentes modificaciones.

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

Después de mucho rato investigando, determiné que la imposiblidad de ejecutar la lambda viene por dos partes:
- El pod de Localstack no puede ejecutar funciones lambda por qué el contenedor no es capaz orquestar contenedores
- El contenedor de la función lambda no es capaz de retornar la respuesta al pod de Localstack y que apunta al dominio del localhost

Para solucionar los fallos, modifqué la chart de Helm, agregando ciertas variables de entorno, creando un nuevo contenedor que hereda del ya configurado pero con el cli de Docker y montando el volumen del socket de Docker.

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

Y con todo eso habilitado, ya es posible invocar a la función lambda sin errores.

```bash
awsl lambda invoke --function-name test --payload '{"test":"value"}' --cli-binary-format raw-in-base64-out output.txt
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

### DynamoDB Streams, Secret Manager y SES

La última pieza por implementar es el envío del correo desde la lambda cuando se produce un evento en DynamoDB. En este caso, he implementado el envío tanto a través de SES como con SMTP. Justo por querer utilizar SMPT, es necesario usar Secret Manager para
gestionar la contraseña del correo que envía el correo, inyectado el valor con una variable de entorno al ejecutar Terraform y recuperando desde la función lambda el secreto con la librería boto3.

```terraform
# Módulo de secret maanger
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

Las variables de entorno de las cuales se va a alimentar Terraform para configurar el valor de los secretos.

```bash
export TF_VAR_sender_email="fake_sender_email@gmail.com"
export TF_VAR_sender_email_secret="fake_password"
export TF_VAR_receiver_email="fake_receiver_email@gmail.com"
```

Para habilitar los DynamoDB streams, hay que modificar los permisos asignados, tanto para que sea capaz de habilitar DynamoDB streams como para gestionar el secreto.

```terraform
# Módulo de lambda
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
            # Permiso para leer un secreto en Secrets Manager
            Effect = "Allow"
            Action = [
                "secretsmanager:GetSecretValue"
            ]
            Resource = var.email_secret_arn
        }
    ]
    })
}

# Vincula un DynamoDB Stream a la Lambda para que se ejecute automáticamente cuando haya cambios en la tabla
resource "aws_lambda_event_source_mapping" "this" {
    event_source_arn = var.table_stream_arn
    function_name    = aws_lambda_function.this.arn
    starting_position = "LATEST"
    enabled = true
}
```

En cuanto al código de Python de la lambda, son dos funciones, cada una con la implementación de SES o SMTP.

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

Con todo esto, ya estaría implementada la lista de servicios que tenía planificados. Tal vez en el futuro expando el proyecto agregando una pipeline para el CI y un chart de Helm con ArgoCD, de momento lo ejo aquí.
