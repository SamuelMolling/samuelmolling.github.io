---
title: "MongoDB Atlas with Terraform"
collection: publications
permalink: /publications/mongodb-atlas-with-terraform
excerpt: 'This article is about how to create a MongoDB Atlas cluster using Terraform.'
date: 2024-01-23
venue: 'MongoDB Developer Center'
# slidesurl: 'http://academicpages.github.io/files/slides1.pdf'
paperurl: 'http://gschmitto.github.io/files/paper1.pdf'
# citation: 'Your Name, You. (2009). &quot;Paper Title Number 1.&quot; <i>Journal 1</i>. 1(1).'
---
In this tutorial, I will show you how to start using MongoDB Atlas with Terraform and create some simple resources. This first part is simpler and more introductory, but in the next article, I will explore more complex items and how to connect the creation of several resources into a single module. The tutorial is aimed at people who want to maintain their infrastructure as code (IaC) in a standardized and simple way. If you already use or want to use IaC on the MongoDB Atlas platform, this article is for you.

What are modules?

They are code containers for multiple resources that are used together. They serve several important purposes in building and managing infrastructure as code, such as:

1. Code reuse.
2. Organization.
3. Encapsulation.
4. Version management.
5. Ease of maintenance and scalability.
6. Sharing in the community.

Everything we do here is contained in the [provider/resource documentation](https://www.mongodb.com/developer/products/atlas/mongodb-atlas-with-terraform/#creating-a-project:~:text=provider/resource%20documentation-,.,-Note%3A%20We%20will)

> Note: We will not use a backend file. However, for productive implementations, it is extremely important and safer to store the state file in a remote location such as an S3, GCS, Azurerm, etc…

## Creating a project

In this first step, we will dive into the process of creating a project using Terraform. Terraform is a powerful infrastructure-as-code tool that allows you to manage and provision IT resources in an efficient and predictable way. By using it in conjunction with MongoDB Atlas, you can automate the creation and management of database resources in the cloud, ensuring a consistent and reliable infrastructure.

To get started, you'll need to install [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)  in your development environment. This step is crucial as it is the basis for running all the scripts and infrastructure definitions we will create. After installation, the next step is to configure Terraform to work with MongoDB Atlas. You will need an API key that has permission to create a project at this time.

To create an API key, you must:

1. Select Access Manager at the top of the page, and click Organization Access.

2. Click Create API Key.

![](/images/mongodb-atlas-with-terraform/image1.png)

3. Enter a brief description of the API key and the necessary permission. In this case, I put it as Organization Owner. After that, click Next.

![](/images/mongodb-atlas-with-terraform/image2.png)

4. Your API key will be displayed on the screen.

![](/images/mongodb-atlas-with-terraform/image3.png)

5. Release IP in the Access List (optional): If you have enabled your organization to use API keys, the requestor's IP must be released in the Access List; you must include your IP in this list. To validate whether it is enabled or not, go to **Organization Settings -> Require IP Access List** for the Atlas Administration API. In my case, it is disabled, as it is just a demonstration, but in case you are using this in an organization, I strongly advise you to enable it.

![](/images/mongodb-atlas-with-terraform/image4.png)

After creating an API key, let's start working with Terraform. You can use the IDE of your choice; I will be using VS Code. Create the files within a folder. The files we will need at this point are:

* **main.tf:** In this file, we will define the main resource, ```mongodbatlas_project```. Here, you will configure the project name and organization ID, as well as other specific settings, such as teams, limits, and alert settings.

* **provider.tf:** This file is where we define the provider we are using — in our case, ```mongodbatlas```. Here, you will also include the access credentials, such as the API key.

* **terraform.tfvars:** This file contains the variables that will be used in our project — for example, the project name, team information, and limits, among others.

* **variable.tf:** Here, we define the variables mentioned in the terraform.tfvars file, specifying the type and, optionally, a default value.

* **version.tf:** This file is used to specify the version of Terraform and the providers we are using.

The main.tf file is the heart of our Terraform project. In it, you start with the data source declaration ```mongodbatlas_roles_org_id``` to obtain the ```org_id```, which is essential for creating the project. Next, you define the ```mongodbatlas_project``` resource with several settings. Here are some examples:

* ```name``` and ```org_id``` are basic settings for the project name and organization ID.

* Dynamic blocks are used to dynamically configure teams and limits, allowing flexibility and code reuse.

* Other settings, like ```with_default_alerts_settings``` and ```is_data_explorer_enabled```, are options for customizing the behavior of your MongoDB Atlas project.

In the main.tf file, we will then add our project resource, called ```mongodbatlas_project```.

```tf
data "mongodbatlas_roles_org_id" "org" {}

resource "mongodbatlas_project" "default" {
 name   = var.name
 org_id = data.mongodbatlas_roles_org_id.org.org_id

 dynamic "teams" {
   for_each = var.teams
   content {
     team_id    = teams.value.team_id
     role_names = teams.value.role_names
   }
 }

 dynamic "limits" {
   for_each = var.limits
   content {
     name  = limits.value.name
     value = limits.value.value
   }
 }

 with_default_alerts_settings                     = var.with_default_alerts_settings
 is_collect_database_specifics_statistics_enabled = var.is_collect_database_specifics_statistics_enabled
 is_data_explorer_enabled                         = var.is_data_explorer_enabled
 is_extended_storage_sizes_enabled                = var.is_extended_storage_sizes_enabled
 is_performance_advisor_enabled                   = var.is_performance_advisor_enabled
 is_realtime_performance_panel_enabled            = var.is_realtime_performance_panel_enabled
 is_schema_advisor_enabled                        = var.is_schema_advisor_enabled
}
```

In the provider file, we will define the provider we are using and the API key that will be used. As we are just testing, I will specify the API key as a variable that we will input into our code. However, when you are using it in production, you will not want to pass the API key in the code in exposed text, so it is possible to pass it through environment variables or even AWS secret manager.

```tf
provider "mongodbatlas" {
   public_key  = var.atlas_public_key
   private_key = var.atlas_private_key
}
```

In the variable.tf file, we will specify the variables that we are waiting for a user to pass. As I mentioned earlier, the API key is an example.

```tf
variable "name" {
 description = <<HEREDOC
   The name of the project you want to create.
   HEREDOC
 type        = string
}

variable "teams" {
 description = <<HEREDOC
   The list of teams that belong to the project.
   The roles can be:
   Organization:
       ORG_OWNER
       ORG_MEMBER
       ORG_GROUP_CREATOR
       ORG_BILLING_ADMIN
       ORG_READ_ONLY
   Project:
       GROUP_CLUSTER_MANAGER
       GROUP_DATA_ACCESS_ADMIN
       GROUP_DATA_ACCESS_READ_ONLY
       GROUP_DATA_ACCESS_READ_WRITE
       GROUP_OWNER
       GROUP_READ_ONLY
   HEREDOC
 type = list(object({
   team_id    = string
   role_names = list(string)
 }))
 default = []
}

variable "is_collect_database_specifics_statistics_enabled" {
 description = <<HEREDOC
   If true, Atlas collects and stores database-specific statistics for the specified project.
   HEREDOC
 type        = bool
 default     = true
}

variable "is_data_explorer_enabled" {
 description = <<HEREDOC
   If true, Atlas enables Data Explorer for the specified project.
   HEREDOC
 type        = bool
 default     = false
}

variable "is_extended_storage_sizes_enabled" {
 description = <<HEREDOC
   If true, Atlas enables extended storage sizes for the specified project.
   HEREDOC
 type        = bool
 default     = true
}

variable "is_performance_advisor_enabled" {
 description = <<HEREDOC
   If true, Atlas enables Performance Advisor for the specified project.
   HEREDOC
 type        = bool
 default     = true
}


variable "is_realtime_performance_panel_enabled" {
 description = <<HEREDOC
   If true, Atlas enables the Real Time Performance Panel for the specified project.
   HEREDOC
 type        = bool
 default     = true
}


variable "is_schema_advisor_enabled" {
 description = <<HEREDOC
   If true, Atlas enables Schema Advisor for the specified project.
   HEREDOC
 type        = bool
 default     = true
}


variable "with_default_alerts_settings" {
 description = <<HEREDOC
   If true, Atlas enables default alerts settings for the specified project.
   HEREDOC
 type        = bool
 default     = true
}


variable "limits" {
 description = <<HEREDOC
   Allows one to configure a variety of limits to a Project. The limits attribute is optional.
   https://www.mongodb.com/docs/atlas/reference/api-resources-spec/v2/#tag/Projects/operation/setProjectLimit
   HEREDOC
 type = list(object({
   name  = string
   value = string
 }))


 default = []
}


variable "atlas_public_key" {
 description = <<HEREDOC
   The public key of the Atlas user you want to use to create the project.
   HEREDOC
 type        = string
}


variable "atlas_private_key" {
 description = <<HEREDOC
   The private key of the Atlas user you want to use to create the project.
   HEREDOC
 type        = string
}
```

As you can see, we are placing several variables. The idea is that we can reuse the module to create different projects that have different specifications. For example, one needs the data from the databases to be visible on the Atlas Platform, so we put the ```is_data_explorer_enabled``` variable as ```True```. On the other hand, if, for some security reason, the company does not want users to use the platform for data visualization, it is possible to specify it as ```False```, and the Collections button that we have when we enter the cluster within Atlas will disappear.

In the versions.tf file, we specify the version of Terraform we use and the necessary providers with their respective versions. This configuration is crucial so that the code always runs in the same version, avoiding inconsistencies and incompatibilities that may arise with updates.

Here is what our providers file will look like:

```tf
terraform {
   required_version = ">= 0.12"
   required_providers {
       mongodbatlas = {
           source = "mongodb/mongodbatlas"
           version = "1.14.0"
       }
   }
}
```

Finally, we have the terraform.tfvars file, where we will pass the variables that we defined in the variable.tf file. Here is an example of how it can be:

* ```required_version = ">= 0.12"```: This line specifies that your Terraform project requires, at a minimum, Terraform version 0.12. By using >=, you indicate that any version of Terraform from 0.12 onward is compatible with your project. This offers some flexibility by allowing team members and automation systems to use newer versions of Terraform as long as they are not older than 0.12.

* ```required_providers```: This section lists the providers required for your Terraform project. In your case, you are specifying the mongodbatlas provider.

* ```source = "mongodb/mongodbatlas"```: This defines the source of the mongodbatlas provider. Here, mongodb/mongodbatlas is the official identifier of the MongoDB Atlas provider in the Terraform Registry.

* ```version = "1.14.0"```: This line specifies the exact version of the mongodbatlas provider that your project will use, which is version 1.14.0. Unlike Terraform configuration, where we specify a minimum version, here you are defining a provider-specific version. This ensures that everyone using your code will work with the same version of the provider, avoiding discrepancies and issues related to version differences.

Finally, we have the variable file that will be included in our code, .tfvars.

```tf
name = "project-test"
atlas_public_key = "YOUR PUBLIC KEY"
atlas_private_key = "YOUR PRIVATE KEY"
```

We are specifying the value of the name variable, which is the name of the project and the public/private key of our provider. You may wonder, "Where are the other variables that we specified in the main.tf and variable.tf files?" The answer is: These variables were specified with a default value within the variable.tf file — for example, the limits value:

```tf
variable "limits" {
 description = <<HEREDOC
   Allows one to configure a variety of limits to a Project. The limits attribute is optional.
   https://www.mongodb.com/docs/atlas/reference/api-resources-spec/v2/#tag/Projects/operation/setProjectLimit
   HEREDOC
 type = list(object({
   name  = string
   value = string
 }))
 default = []
}
```

We are saying that if no information is passed in .tfvars, the default value is empty. This means that it will not create any limit rules for our project. If we want to specify a limit, we just put the following variable in .tfvars:

```tf
name              = "project-test"
atlas_public_key  = "YOUR PUBLIC KEY"
atlas_private_key = "YOUR PRIVATE KEY"
limits = [{
 "name" : "atlas.project.deployment.clusters",
 "value" : 26
 }, {
 "name" : "atlas.project.deployment.nodesPerPrivateLinkRegion",
 "value" : 51
}]
```

Now is the time to apply. =D

We run a ```terraform init``` in the terminal in the folder where the files are located so that it downloads the providers, modules, etc…

Now that init has worked, let's run the plan and evaluate what will happen. You can run the plan, like this ```terraform plan```:

```bash
samuelmolling@Samuels-MacBook-Pro project % terraform plan
data.mongodbatlas_roles_org_id.org: Reading...
data.mongodbatlas_roles_org_id.org: Read complete after 0s [id=5d7072c9014b769c4bd89f60]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
   + create

Terraform will perform the following actions:

   # mongodbatlas_project.default will be created
   + resource "mongodbatlas_project" "default" {
       + cluster_count = (known after apply)
       + created = (known after apply)
       + id = (known after apply)
       + is_collect_database_specifics_statistics_enabled = true
       + is_data_explorer_enabled = false
       + is_extended_storage_sizes_enabled = true
       + is_performance_advisor_enabled = true
       + is_realtime_performance_panel_enabled = true
       + is_schema_advisor_enabled = true
       + name = "project-test"
       + org_id = "5d7072c9014b769c4bd89f60"
       + with_default_alerts_settings = true

       + limits {
           + current_usage = (known after apply)
           + default_limit = (known after apply)
           + maximum_limit = (known after apply)
           + name = "atlas.project.deployment.clusters"
           + value = 26
         }
       + limits {
           + current_usage = (known after apply)
           + default_limit = (known after apply)
           + maximum_limit = (known after apply)
           + name = "atlas.project.deployment.nodesPerPrivateLinkRegion"
           + value = 51
         }
     }

Plan: 1 to add, 0 to change, 0 to destroy.

───────────────────────────────────────── ───────── ───────────────────────────────────────── ───────── ───────────────────────────────────────── ───────── ───────────────────────────────────────── ────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run `terraform apply` now.
```

Show! It was exactly the output we expected to see: the creation of a project resource, with some limits and ```is_data_explorer_enabled``` set to False. Let's apply this!

When you run the command ```terraform apply```, you will be asked for approval with ```yes``` or ```no```. Type ```yes```.

```bash
samuelmolling@Samuels-MacBook-Pro project % terraform apply
data.mongodbatlas_roles_org_id.org: Reading...
data.mongodbatlas_roles_org_id.org: Read complete after 0s [id=5d7072c9014b769c4bd89f60]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
   + create

Terraform will perform the following actions:

   # mongodbatlas_project.default will be created
   + resource "mongodbatlas_project" "default" {
       + cluster_count = (known after apply)
       + created = (known after apply)
       + id = (known after apply)
       + is_collect_database_specifics_statistics_enabled = true
       + is_data_explorer_enabled = false
       + is_extended_storage_sizes_enabled = true
       + is_performance_advisor_enabled = true
       + is_realtime_performance_panel_enabled = true
       + is_schema_advisor_enabled = true
       + name = "project-test"
       + org_id = "5d7072c9014b769c4bd89f60"
       + with_default_alerts_settings = true

       + limits {
           + current_usage = (known after apply)
           + default_limit = (known after apply)
           + maximum_limit = (known after apply)
           + name = "atlas.project.deployment.clusters"
           + value = 26
         }
       + limits {
           + current_usage = (known after apply)
           + default_limit = (known after apply)
           + maximum_limit = (known after apply)
           + name = "atlas.project.deployment.nodesPerPrivateLinkRegion"
           + value = 51
         }
     }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
   Terraform will perform the actions described above.
   Only 'yes' will be accepted to approve.

   Enter a value: yes

mongodbatlas_project.default: Creating...
mongodbatlas_project.default: Creation complete after 9s [id=659ed54eb3343935e840ce1f]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Now, let's look at Atlas to see if the project was created successfully...

![](/images/mongodb-atlas-with-terraform/image5.png)

It worked!

In this tutorial, we saw how to create our first API key. We created a project using Terraform and our first module. In an upcoming article, we’ll look at how to create a cluster and user using Terraform and Atlas.

To learn more about MongoDB and various tools, I invite you to enter the 
[Developer Center](https://www.mongodb.com/developer/products/atlas/mongodb-atlas-with-terraform/#creating-a-project:~:text=It%20worked!,the%20other%20articles.) to read the other articles.
