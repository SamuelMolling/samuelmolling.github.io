---
title: "MongoDB Atlas With Terraform: Database Users and Vault"
collection: publications
permalink: /publications/mongodb-atlas-with-terraform-database-users-and-vault
excerpt: 'In this tutorial, I will show how to create a user for the MongoDB database in Atlas using Terraform and how to store this credential securely in HashiCorp Vault.'
date: 2024-04-15
venue: 'MongoDB Developer Center'
paperurl: 'https://www.mongodb.com/developer/products/atlas/mongodb-atlas-terraform-database-users-vault/'
---
In this tutorial, I will show how to create a user for the MongoDB database in Atlas using Terraform and how to store this credential securely in HashiCorp Vault. We saw in the previous article, 
MongoDB Atlas With Terraform - Cluster and Backup Policies(https://gschmitto.github.io/publications/mongodb-atlas-with-terraform-cluster-and-backup-policies), how to create a cluster with configured backup policies. Now, we will go ahead and create our first user. If you haven't seen the previous articles, I suggest you look to understand how to get started.

This article is for anyone who intends to use or already uses infrastructure as code (IaC) on the MongoDB Atlas platform or wants to learn more about it.

Everything we do here is contained in the provider/resource documentation:

* [mongodbatlas_database_user](https://registry.terraform.io/providers/mongodb/mongodbatlas/latest/docs/resources/database_user)
* [vault_kv_secret_v2](https://registry.terraform.io/providers/hashicorp/vault/latest/docs/resources/kv_secret_v2)

> Note: We will not use a backend file. However, for productive implementations, it is extremely important and safer to store the state file in a remote location such as an S3, GCS, Azurerm, etc…

## Creating a User
At this point, we will create our first user using Terraform in MongoDB Atlas and store the URI to connect to my cluster in HashiCorp Vault. For those unfamiliar, [HashiCorp Vault](https://www.hashicorp.com/products/vault) is a secrets management tool that allows you to securely store, access, and manage sensitive credentials such as passwords, API keys, certificates, and more. It is designed to help organizations protect their data and infrastructure in complex, distributed IT environments. In it, we will store the connection URI of the user that will be created with the cluster we created in the [last article](https://gschmitto.github.io/publications/mongodb-atlas-with-terraform-cluster-and-backup-policies).

Before we begin, make sure that all the prerequisites mentioned in the previous article are properly configured: Install [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli), create an API key in MongoDB Atlas, and set up a project and a cluster in Atlas. These steps are essential to ensure the success of creating your database user.

## Configuring HashiCorp Vault to run on Docker

The first step is to run HashiCorp Vault so that we can test our module. It is possible to run Vault on Docker Local. If you don't have Docker installed, you can [download it](https://docs.docker.com/engine/install/). After downloading Docker, we will download the image we want to run — in this case, from Vault. To do this, we will execute a command in the terminal ```docker pull vault:1.13.3``` or download using Docker Desktop.

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image1.png)

Now, we will create a container from this image. Click on the image and click on **Run**. After this, a box will open where we only need to map the port from our computer to the container. In this case, I will use port 8200 which is the Vault's default port. Click **Run**.

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image2.png)

The container will start running. If we go to our browser and enter the URL ```localhost:8200/```, the Vault login screen will appear.

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image3.png)

To access the Vault, we will use the Root Token that is generated when we create the container.

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image4.png)

Now, we will log in. After opening, we will create a new KV-type engine just to illustrate it a little better. Click **Secrets Engines -> Enable new Engine -> Generic KV** and click **Next**.

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image5.png)

In Path, put ```kv/my_app``` and click on **Enable Engine**. Now, we have our Vault configured and working.

## Terraform provider configuration for MongoDB Atlas and HashiCorp Vault

The next step is to configure the Terraform provider. This will allow Terraform to communicate with the MongoDB Atlas and Vault API to manage resources. Add the following block of code to your providers.tf file:

```tf
provider "mongodbatlas" {}
provider "vault" {
  address = "http://localhost:8200"
  token = "hvs.brmNeZd31NwEmyky1uYI2wvY"
  skip_child_token = true
}
```

In the previous article, we configured the Terraform provider by placing our public and private keys in environment variables. We will continue in this way. We will add a new provider, the Vault. In it, we will configure the Vault address, the authentication token, and the skip_child_token parameter so that we can authenticate to the Vault.

> Note: It is not advisable to specify the authentication token in a production environment. Use one of the authentication methods recommended by HashiCorp, such as app_role. You can evaluate the options in [Terraform’s docs](https://registry.terraform.io/providers/hashicorp/vault/latest/docs)

## Creating the Terraform version file

The version file continues to have the same purpose, as mentioned in other articles, but we will add the version of the Vault provider as something new.

```tf
terraform {
    required_version = ">= 0.12"
    required_providers {
        mongodbatlas = {
            source = "mongodb/mongodbatlas"
            version = "1.14.0"
        }
        vault = {
            source = "hashicorp/vault"
            version = "4.0.0"
        }
    }
}
```

### Defining the database user and vault resource

After configuring the version file and establishing the Terraform and provider versions, the next step is to define the user resource in MongoDB Atlas. This is done by creating a .tf file — for example, main.tf — where we will create our module. As we are going to make a module that will be reusable, we will use variables and default values so that other calls can create users with different permissions, without having to write a new module.

```tf
# ------------------------------------------------------------------------------
# RANDOM PASSWORD
# ------------------------------------------------------------------------------
resource "random_password" "default" {
    length = var.password_length
    special = false
}

# ------------------------------------------------------------------------------
# DATABASE USER
# ------------------------------------------------------------------------------
resource "mongodbatlas_database_user" "default" {
    project_id = data.mongodbatlas_project.default.id
    username = var.username
    password = random_password.default.result
    auth_database_name = var.auth_database_name

    dynamic "roles" {
        for_each = var.roles
        content {
            role_name = try(roles.value["role_name"], null)
            database_name = try(roles.value["database_name"], null)
            collection_name = try(roles.value["collection_name"], null)
        }
    }

    dynamic "scopes" {
        for_each = var.scope
        content {
            name = scopes.value["name"]
            type = scopes.value["type"]
        }
    }


    dynamic "labels" {
        for_each = local.tags
        content {
            key = labels.key
            value = labels.value
        }
    }
}

resource "vault_kv_secret_v2" "default" {
    mount = var.vault_mount
    name = var.secret_name
    data_json = jsonencode(local.secret)
}
```

At the beginning of the file, we have the random_password resource that is used to generate a random password for our user. In the mongodbatlas_database_user resource, we will specify our user details. We are placing some values as variables as done in other articles, such as name and auth_database_name with a default value of admin. Below, we create three dynamic blocks: roles, scopes, and labels. For roles, it is a list of maps that can contain the name of the role (read, readWrite, or some other), the database_name, and the collection_name. These values can be optional if you create a user with atlasAdmin permission, as in this case, it does not. It is necessary to specify a database or collection, or if you wanted, to specify only the database and not a specific collection. We will do an example. For the scopes block, the type is a DATA_LAKE or a CLUSTER. In our case, we will specify a cluster, which is the name of our created cluster, the demo cluster. And the labels serve as tags for our user.

Finally, we define the vault_kv_secret_v2 resource that will create a secret in our Vault. It receives the mount where it will be created and the name of the secret. The data_json is the value of the secret; we are creating it in the locals.tf file that we will evaluate below. It is a JSON value — that is why we are encoding it.

In the variable.tf file, we create variables with default values:

```tf
variable "project_name" {
    description = "The name of the Atlas project"
    type = string
}

variable "cluster_name" {
    description = "The name of the Atlas cluster"
    type = string
}

variable "password_length" {
    description = "The length of the password"
    type = number
    default = 20
}

variable "username" {
    description = "The username of the database user"
    type = string
}

variable "auth_database_name" {
    description = "The name of the database in which the user is created"
    type = string
    default = "admin"
}

variable "roles" {
    description = <<HEREDOC
    Required - One or more user roles blocks.
    HEREDOC
    type = list(map(string))
}

variable "scope" {
    description = "The scopes to assign to the user"
    type = list(object({
        name = string
        type = string
    }))
    default = []
}

variable "labels" {
    type = map(any)
    default = null
    description = "A JSON containing additional labels"
}

variable "uri_options" {
    type = string
    default = "retryWrites=true&w=majority&readPreference=secondaryPreferred"
    description = "A string containing additional URI configs"
}

variable "vault_mount" {
    description = "The mount point for the Vault secret"
    type = string
}

variable "secret_name" {
    description = "The name of the Vault secret"
    type = string
}

variable "application" {
    description = <<HEREDOC
    Optional - Key-value pairs that tag and categorize the cluster for billing and organizational purposes.
    HEREDOC
    type = string
}

variable "environment" {
    description = <<HEREDOC
    Optional - Key-value pairs that tag and categorize the cluster for billing and organizational purposes.
    HEREDOC
    type = string
}
```

We configured a file called **locals.tf** with the values for our Vault and the tags that were created, like the last article. The interesting thing here is that we are defining how our user's connection string will be assembled and saved in the Vault. We could only save the username and password too, but I personally prefer to save the URI. This way, I can specify some good practices such as defining connection tags, such as readPreference, without depending on the developer to place it in the application. In the code below, there are some text treatments so that the URI is correct. At the end, I create a variable called **secret** that has a URI key and receives the value of the created URI.

```tf
locals {
    private_connection_srv = data.mongodbatlas_advanced_cluster.default.connection_strings.0.standard_srv
    cluster_uri = trimprefix(local.private_connection_srv, "mongodb+srv://")
    private_connection_string = "mongodb+srv://${mongodbatlas_database_user.default.username}:${random_password.default.result}@${local.cluster_uri}/${var.auth_database_name}?${var.uri_options}"

    secret = { "URI" = local.private_connection_string }

    tags = {
    name = var.application
    environment = var.environment
    }
}
```

In this article, we adopt the use of data sources in Terraform to establish a dynamic connection with existing resources, such as our MongoDB Atlas project and our cluster. Specifically, in the data.tf file, we define a data source, mongodbatlas_project and mongodbatlas_advanced_cluster, to access information about an existing project and cluster based on its name:

```tf
data "mongodbatlas_project" "default" {
    name = var.project_name
}


data "mongodbatlas_advanced_cluster" "default" {
    project_id = data.mongodbatlas_project.default.id
    name = var.cluster_name
}
```

Finally, we define our variables file, terraform.tfvars:

```tf
project_name = "project-test"
username = "usr_myapp"
application = "teste-cluster"
environment = "dev"
cluster_name = "cluster-demo"

roles = [{
        "role_name" = "readWrite",
        "database_name" = "db1",
        "collection_name" = "collection1"
    }, {
        "role_name" : "read",
        "database_name" : "db2"
}]

scope = [{
    name = "cluster-demo",
    type = "CLUSTER"
}]

secret_name = "MY_MONGODB_SECRET"
vault_mount = "kv/my_app"
```

These values defined in terraform.tfvars are used by Terraform to populate corresponding variables in your configuration. In it, we are specifying the user's scope, values for the Vault, and our user's roles. The user will have readWrite permission on db1 in collection1 and read permission on db2 in all collections for the demo cluster.

The file structure is as follows:

* **main.tf:** In this file, we will define the main resource, the **mongodbatlas_database_user** and **vault_kv_secret_v2**, along with a random password generation. Here, you have configured the cluster and backup routines.

* **provider.tf:** This file is where we define the provider we are using, in our case, mongodbatlas and Vault.

* **terraform.tfvars:** This file contains the variables that will be used in our module — for example, the user name and Vault information, among others.

* **variable.tf:** Here, we define the variables mentioned in the terraform.tfvars file, specifying the type and, optionally, a default value.

* **version.tf:** This file is used to specify the version of Terraform and the providers we are using.

* **data.tf:** Here, we specify a datasource that will bring us information about our project and created cluster. We will search for its name and for our module, it will give us the project ID and cluster information such as its connection string.

* **locals.tf:** We specify example tags to use in our user and treatments to create the URI in the Vault.

Now is the time to apply. =D

We run a Terraform init in the terminal in the folder where the files are located so that it downloads the providers, modules, etc…

> Note: Remember to export the environment variables with the public and private key.

```sh
export MONGODB_ATLAS_PUBLIC_KEY="your_public_key"
export MONGODB_ATLAS_PRIVATE_KEY="your_private_key"
```

Now, we run init and then plan, as in previous articles.

We assess that our plan is exactly what we expect and run the apply to create it.

When running the ```terraform apply``` command, you will be prompted for approval with ```yes``` or ```no```. Type ```yes```.

Now, let's look in Atlas to see if the user was created successfully...

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image6.png)

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image7.png)

Let's also look in the Vault to see if our secret was created.

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image8.png)

It was created successfully! Now, let's test if the URI is working perfectly.

This is the format of the URI that is generated: ```mongosh "mongodb+srv://usr_myapp:<password>@<clusterEndpoint>/admin?retryWrites=true&majority&readPreference=secondaryPreferred"```

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image9.png)

Success! Now, in db3, make sure it will not have permission in another database.

![](/images/mongodb-atlas-with-terraform-database-users-and-vault/image10.png)

Excellent — permission denied, as expected.

We have reached the end of this series of articles about MongoDB. I hope they were enlightening and useful for you!

To learn more about MongoDB and various tools, I invite you to visit the 
[Developer Center](https://www.mongodb.com/developer/) to read the other articles.