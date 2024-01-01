# Service Discovery: Register your service with Consul

## Overview

Consul is a service discovery tool that allows services to discover and communicate with each other automatically.

Services deployed as ECS instances are immediately added to Consul’s catalog of existing services. When an ECS instance spins up, a Registrator and Consul Agent will be created within the cluster, ensuring that there is always one Registrator and one Agent per ECS instance.

The Registrator monitors the Docker daemon for container stop and start events and handles service registration with Consul, using the container names and exposed ports as the service information (which you will explicitly provide, as shown below).

Lambdas, on the other hand, will be manually added to Consul’s registry.

> [!IMPORTANT]
> At this time, service discovery is only available for *internal* services.

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

Replace `<replace_me>` with the url for dev consul. You can find the urls for dev, qa, and prod [here]().

## ECS Services

In order to register your ECS service with Consul, you'll add the following environment variables to your ECS task definition.

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

To do so, you will need to:

* retrieve the `consul-client` remote state.
* define the lambda node address, your service tags, and health check within a `consul_service` resource.

```
data "terraform_remote_state" "consul-client" {
    backend = "s3"
    config  = {
        bucket  = "rev-s3-use1-ss-terraform-{{ TF_VAR_env }}-infra"
        key     = "{{ TF_VAR_env }}/{{ TF_VAR_region_short }}/consul-client/{{ TF_VAR_env }}.tfstate"
        profile = "rev ss"
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
> Make sure to define `node`, `tags`, and `check` exactly as shown in the example above, or your lambda will fail to register.

## Verify that your registration was successful

Make a request to the Consul server for your service:

`https://discovery.dev.rev.io/<your_service_name>/<env>/<endpoint>`

You must include `env` in the url path as seen above. This will take on the value dev, qa, or prod.

> [!TIP]
> An easy way to verify your service's registration is to make a request to a health check endpoint that simply returns a 200 OK JSON response and success message.

## Load Balancing

Consul/Fabio take care of load balancing automatically. They will route requests to your service in a round-robin (rotating) manner, depending on how many instances of your service are available at a given time. 

For example, if there are two instances of a service up and running, Consul will route 50% of requests to one instance and 50% to the other instance. If there are three instances, Consul will route 33% of requests to each instance.

## Nice to have

If possible, please set a `User-Agent` header for your service's name in your requests. This will allow for more comprehensive logging in the future.

## Troubleshooting

If you run into any issues or have any questions, check out our [troubleshooting]() doc. If you're still having a hard time, feel free to reach out in the `#eng-consul` slack channel!
