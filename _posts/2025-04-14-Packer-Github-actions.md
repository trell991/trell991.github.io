---
title: "Building Packer images and uploading to Azure via Github Actions"
date: 2025-04-15 8:35:00 +0000
categories: [Personal Projects]
tags: [Packer,Terraform,Github Actions]
layout: post
---

## Packer + Github Actions

When using virtual machines for hosting applications Packer can be an effective tool for creating and provisioning the VM.
Normally I will build packer images via the commandline, upload them to Azure then deploy instances of that image with terraform.

Then I thought to myself why not automate this process with a pipeline tool? So off I went digging into the Packer and Github Actions pipeline built out to smoothen out my development proccess.

After may attempts and failures (69 builds) I was able to get the pipeline working with Packer. This pipeline is apart of a larger project im working on called **Chaos Labs** but I wanted to share this pipeline flow because I ran into a couple of nuanced issue while building it

## Packer Directory structure

First, let's dive into the Packer directory structure within your project. I like to organize my Packer configurations inside the `build` directory. Within `build`, I create a `packer` subdirectory, as shown in the diagram below.

> I'm using mermaid to make these snazzy diagrams
{: .prompt-tip }

<div class="mermaid">
graph TD
    A[packer]
    A --> B[golden-grafana]
    B --> C[golden-grafana-vars.auto.pkrvars.hcl]
    B --> D[ubuntu-server-jammy.pkr.hcl]
</div>

Within the `packer` subdirectory, I then create a directory for the image I'm building. In this example, I've named the image `golden-grafana`. Inside the `golden-grafana` directory are my Packer files: `golden-grafana-vars.auto.pkrvars.hcl` and `ubuntu-server-jammy.pkr.hcl`.

## Lets see the code

Next lets take a look at the code to see how all of this is working
![alt text](https://media1.tenor.com/m/WKCshDvPdCAAAAAC/alright-what%27s-the-code-wesley-chan.gif)

### Packer script

Ok lets take a look at my Packer script. Within my script im defining all of my variables at the top portion of the script see below:

```hcl
packer {
  required_plugins {
    azure = {
      source  = "github.com/hashicorp/azure"
      version = "~> 2"
    }
  }
}

variable "subscription_id" {
  type    = string
  default = null
}

variable "managed_image_name" {
  type = string
}

variable "managed_image_resource_group_name" {
  type = string
}

variable "os_type" {
  type = string
}

variable "image_publisher" {
  type = string
}

variable "image_offer" {
  type = string
}

variable "image_sku" {
  type = string
}

variable "location" {
  type = string
}

variable "vm_size" {
  type = string
}
variable "communicator" {
  type = string
}

variable "ssh_username" {
  type = string
}

variable "ssh_private_key_file" {
  type = string
}

variable "ssh_timeout" {
  type = string
}

variable "os_disk_size_gb" {
  type = number
}

# Source: azure-arm builder
source "azure-arm" "ubuntu-jammy" {
  use_azure_cli_auth                = true
  communicator                      = "ssh"
  location                          = var.location
  image_publisher                   = var.image_publisher
  image_offer                       = var.image_offer
  image_sku                         = var.image_sku
  vm_size                           = var.vm_size
  os_disk_size_gb                   = var.os_disk_size_gb
  ssh_username                      = var.ssh_username
  ssh_private_key_file              = var.ssh_private_key_file
  os_type                           = var.os_type
  managed_image_name                = var.managed_image_name
  managed_image_resource_group_name = var.managed_image_resource_group_name
}

build {
  sources = ["source.azure-arm.ubuntu-jammy"]

  # Provisioning with shell scripts
  provisioner "shell" {
    scripts = [
      "../../scripts/tooling.sh",
      "../../scripts/monitoring-stack.sh"
    ]
  }

  # Provisioning with shell scripts inline
  provisioner "shell" {
    inline = [
      "echo 'Updating system packages'",
      "sudo apt-get update",
      "sudo apt-get upgrade -y",
      "echo 'Cloning repo'",
      "git clone https://github.com/Poupmediagroup/Chaos-labs.git"
    ]
  }

  # Post-processors to show final output
  post-processors {
    post-processor "shell-local" {
      inline = [
        "echo '=== Packer Build Outputs ==='",
        "echo \"Image Name: ${var.managed_image_name}\"",
        "echo \"Resource Group: ${var.managed_image_resource_group_name}\""
      ]
    }
  }
}

```

I'm structuring my Packer project this way because I was running into a lot of issues with Packer not being able to find my variable declarations when running the workflow in GitHub Actions. As you can see in the Packer script, I'm declaring all the variables required to build the image and setting the `subscription_id` variable to `null`, since that value will be passed in from GitHub Actions at runtime.

Now lets take a look at the `vars.auto.pkvars.hcl` file

```hcl
subscription_id                   = null # Will be injected securely from GitHub Secrets
managed_image_name                = "golden-grafana"
os_type                           = "Linux"
image_publisher                   = "Canonical"
image_offer                       = "0001-com-ubuntu-server-jammy"
image_sku                         = "22_04-lts"
location                          = "eastus"
vm_size                           = "Standard_DS2_v2"
communicator                      = "ssh"
ssh_username                      = "test-user"
ssh_private_key_file              = null # Will be injected securely from GitHub Secrets
ssh_timeout                       = "10m"
os_disk_size_gb                   = 30
managed_image_resource_group_name = "your-resource-group-name"
```

Normally, I use a `variables.pkr.hcl` file for declaring variables when building images from the command line with Packer. However, within a build pipeline, using a file with the `.auto.pkrvars.hcl` extension makes incorporating Packer builds into GitHub Actions much easier. Packer automatically loads any `.auto.pkrvars.hcl` files, which simplifies the workflow and reduces the need to manually pass each variable during execution.

### Pipeline YAML

Lastly lets break down the YAML for this workflow

#### Runner and Authentication

In the YAML snippet below, I am specifying the runner I want to use, setting the environment, cloning my project code onto the runner, and using the Azure CLI to authenticate with my Azure tenant.

```yaml
name: packer

on:
  push:
    branches:
        - main
        
env:
  PRODUCT_VERSION: "latest"

jobs:
  packer:
    permissions:
      id-token: write
    runs-on: ubuntu-latest
    name: Build golden-grafana
    environment: dev
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure CLI Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

Next in the snippet below im setting up packer on the runner and initializing packer

```yaml
- name: Setup `packer`
        uses: hashicorp/setup-packer@main
        id: setup
        with:
          version: ${{ env.PRODUCT_VERSION }}

      - name: Run `packer init`
        id: init
        run: "packer init ./build/packer/golden-grafana/ubuntu-server-jammy.pkr.hcl"
```

Next, Packer needs an SSH connection to connect to the virtual machine in order to start the provisioning process. However, an .ssh directory does not exist on the GitHub Actions runner by default (I discovered this through trial and error). In the snippet below, I run an inline shell script to create the .ssh directory and add my private key, which I'm storing in GitHub Actions as a secret. For improved security, a secrets manager such as HashiCorp Vault could be used instead.
Lastly, the script sets the appropriate file permissions so that the SSH key can be read correctly by Packer.

```yaml
- name: Set up packer ssh key
        run: |
          echo "Creating ssh directory..."
          mkdir -p ~/.ssh
          echo "${{ secrets.PACKER_GRAFANA_KEY }}" > $HOME/.ssh/packer_key
          chmod 600 ~/.ssh/packer_key
          echo "Added ssh key and set permissions"

      - name: Validate packer config
        run: |
          echo "Validating packer build"
          cd build/packer/golden-grafana
          packer validate .
```

Lastly, it's time to build the image. In the snippet below, I'm passing in my secrets and variable values using the -var parameter in Packer. I also include my Packer template file, `ubuntu-server-jammy.pkr.hcl`, at the end of the command. After this step, we have a fully working GitHub Actions workflow to build a Packer image!

```yaml
- name: Build image
        id: build
        run: |
          echo "Building packer image"
          cd build/packer/golden-grafana
          packer build \
            -var="ssh_private_key_file=${HOME}/.ssh/packer_key" \
            -var="subscription_id=${{ secrets.AZURE_SUBSCRIPTION_ID }}" \
            -var-file="golden-grafana-vars.auto.pkrvars.hcl" \
            ubuntu-server-jammy.pkr.hcl
```

<script type="module">
 import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
 mermaid.initialize({
  startOnLoad: true,
  theme: 'dark'
 });
</script>
