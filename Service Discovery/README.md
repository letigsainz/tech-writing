# Service Discovery: Register your service with Consul

## Overview

Services deployed as ECS instances are automagically added to Consul’s catalog of existing services. When an ECS instance spins up, a new Registrator and Consul Agent will be created within the cluster, ensuring that there is always one Registrator and one Agent per ECS instance.

The Registrator monitors the Docker daemon for container stop and start events and handles service registration with Consul, using the container names and exposed ports as the service information (which you will explicitly provide).

Lambdas need to be “manually” added to Consul’s registry, as you'll see in this guide.

> [!IMPORTANT]
> At this time, service discovery is only available for internal services.

## Getting Started

Declare the consul provider in your `providers.tf` file.

```
provider "aws" {
    region  = var.default_region
    version = "~> 3.8"
    assume_role {
        role_arn = arn:aws:iam::${var.aws_account_number}:role/GitlabCICDRole
    }
}

provider "consul" {
    address = "${var.consul_server_leader_address}:8500"
    datacenter = "us-east-1"
    version = "2.12.0"
}
```

The variable `consul_server_leader_address` should be declared in your *.vars files.

```
variable "consul_server_leader_address" {
    default = "<replace_me>"
}
```

You will need the addresses/url's for consul servers in dev, qa, and prod. You can find these [here]().
Replace `<replace_me>` with the url for dev consul.

## ECS Services

In order to register your ECS service with Consul, you'll add a few environment variables to your ECS task definition.

* Your service's name.
* Service tags.
* A health check endpoint, interval, and timeout.

```
data "template_file" "your_service_task_definitions" {
    template = file("${path.module}/taskdefinition.json.tpl")
    vars = {
        
        ...
        
        # service details
        { name: "SERVICE_NAME", value: "your_service_name" },
        
        # consul tags
        { name: "SERVICE_TAGS", value: "urlprefix-/${var.project_name}/${var.env} strip=/${var.project_name}/${var.env}" },

        # consul health check
        { name: "SERVICE_CHECK_HTTP", value: "/health-check/" },
        { name: "SERVICE_CHECK_INTERVAL", value: "5s" },
        { name: "SERVICE_CHECK_TIMEOUT", value: "2s" }
    }
}
```

## Lambda Services

All lambda services should be registered to the existing Lambda Node in Consul. 

To register your lambda, you will need to:

* retrieve the `consul-client` remote state.
* define the lambda node address, your service tags, and health check within a `consul_service` resource.

```
data "terraform_remote_state" "consul-client" {
    backend = "s3"
    config  = {
        bucket  = "revup-s3-use1-ss-terraform-{{ TF_VAR_env }}-infra"
        key     = "{{ TF_VAR_env }}/{{ TF_VAR_region_short }}/consul-client/{{ TF_VAR_env }}.tfstate"
        profile = "revup ss"
        region  = "us-east-1"
    }
}

locals {
    invoke_url_split_list  = split("/", aws_api_gateway_deployment.lambda_API_deployment.invoke_url)
    fqdn                   = local.invoke_url_split_list[2]  # cpa2n8k27.execute-api.us-east-1.amazonaws.com
}

resource "consul_service" "lambda-test" {
    service_id  = var.project_name
    name        = var.project_name
    node        = data.terraform_remote_state.consul-client.outputs.lambda-node-name
    address     = local.fqdn
    port        = 443
    tags        = ["url-prefix-/${var.project_name} proto=https host=${local.fqdn} strip=/${var.project_name}"]
    depend_on   = [aws_api_gateway_deployment.lambda_API_deployment]

    check {
        check_id  = "<your_service_name>"  # this must be UNIQUE per lambda
        name      = "Lambda Fake health check"
        status    = "passing"
        interval  = "999999999s"
        timeout   = "999999999s"
    }
}
```

> [!IMPORTANT]
> Make sure to keep the `node`, `tags`, and `check` exactly as shown in the example above, in order for your lambda to get successfuly registered.

## Verify that your registration was successful

Make a request to the Consul server for your service:

`https://discovery.dev.revup.io/<your_service_name>/<env>/<endpoint>`


## Nice to have

If possible, please set a `User-Agent` header for your service's name in your requests. This will allow for more comprehensive logging.