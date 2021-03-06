# ASG DNS handler

## Purpose

This Terraform module sets up everything necessary for dynamically setting
host names following a certain pattern on instances spawned by AWS Auto Scaling
Groups (ASGs).

Learn more about our motivation to build this module in [this blog post](https://underthehood.meltwater.com/blog/2020/02/07/dynamic-route53-records-for-aws-auto-scaling-groups-with-terraform/).

## Requirements

- [Terraform](https://www.terraform.io/downloads.html) 0.12+
- [Terraform AWS provider](https://github.com/terraform-providers/terraform-provider-aws) 2.0+

## Usage

### Per instance names

Create an ASG and set the `asg:hostname_pattern` tag for example like this:

```
asg-test-#instanceid.example@Z3QP9GZSRL8IVA
   ^          ^         ^             ^
   |          |         |           Zone ID to attach routes too
   |          |        Any subdomains that should be added (var.vpc_name)
   |      Replaced by the instance ID
   | Static prefix common across all instances (var.hostname_prefix)
```

Could be interpolated in Terraform like this:

```hcl
tag {
  key                 = "asg:hostname_pattern"
  value               = format("%s-#instanceid.%s@%s", var.hostname_prefix, var.vpc_name, var.internal_zone_id)
  propagate_at_launch = true
}
```

### Single DNS name for the entire ASG

You may optionally use `var.hostname_prefix` alone for _all_ instances in the ASG.

You can enable this by setting `multi_host = true` in the module definition.

```hcl
module "autoscale_dns" {
  # <...>

  multi_host = true
}

```

Following from the examples above we omit the `-#instanceid` portion. eg
`asg-test.example@ABCDEFGHIJ123`.

**Constraints of running multi_host**

If multiple events are happening in quick succession we may get into situations
where latter runs pickup instances that have not finished terminating.

In situations were multiple terminations are expected it may be better to change the logic from

- Scanning the ASG and building the IP list from there

to

- Grabbing the IP list from EC2 and stripping out the instance. This may require
  more information to be stored in TXT entries to map instance ID's to IP addresses

In general it is expected that on busy ASG's there will be residual IP's between
scaling events

```hcl
tag {
  key                 = "asg:hostname_pattern"
  value               = format("%s.%s@%s", var.hostname_prefix, var.vpc_name, var.internal_zone_id)
  propagate_at_launch = true
}
```

### Common

Once you have your ASG set up, you can just invoke this module and point to it:
`use_public_ip` defaults to false

```hcl
module "clever_name_autoscale_dns" {
  source  = "meltwater/asg-dns-handler/aws"
  version = "x.y.z"
  # use_public_ip = true
  autoscale_handler_unique_identifier = "clever_name"
  autoscale_route53zone_arn           = "ABCDEFGHIJ123"
  vpc_name                            = "example"
}
```

## How does it work?

The module sets up these things:

1. A SNS topic
2. A Lambda function
3. A topic subscription sending SNS events to the Lambda function

The Lambda function then does the following:

- Fetch the `asg:hostname_pattern` tag value from the ASG, and parse out the hostname and Route53 zone ID from it.
- If it's an instance being **created**
  - Fetch internal IP from EC2 API
  - Create a Route53 record pointing the hostname to the IP
  - Set the Name tag of the instance to the initial part of the generated hostname
- If it's an instance being **deleted**
  - Fetch the internal IP from the existing record from the Route53 API
  - Delete the record

## Setup

Add `initial_lifecycle_hook` definitions to your `aws_autoscaling_group` resource , like so:

```hcl
resource "aws_autoscaling_group" "my_asg" {
  name = "myASG"

  vpc_zone_identifier = var.aws_subnets

  min_size                  = var.asg_min_count
  max_size                  = var.asg_max_count
  desired_capacity          = var.asg_desired_count
  health_check_type         = "EC2"
  health_check_grace_period = 300
  force_delete              = false

  launch_configuration = aws_launch_configuration.my_launch_config.name

  lifecycle {
    create_before_destroy = true
  }

  initial_lifecycle_hook {
    name                    = "lifecycle-launching"
    default_result          = "ABANDON"
    heartbeat_timeout       = 60
    lifecycle_transition    = "autoscaling:EC2_INSTANCE_LAUNCHING"
    notification_target_arn = module.autoscale_dns.autoscale_handling_sns_topic_arn
    role_arn                = module.autoscale_dns.agent_lifecycle_iam_role_arn
  }

  initial_lifecycle_hook {
    name                    = "lifecycle-terminating"
    default_result          = "ABANDON"
    heartbeat_timeout       = 60
    lifecycle_transition    = "autoscaling:EC2_INSTANCE_TERMINATING"
    notification_target_arn = module.autoscale_dns.autoscale_handling_sns_topic_arn
    role_arn                = module.autoscale_dns.agent_lifecycle_iam_role_arn
  }

  tag {
    key                 = "asg:hostname_pattern"
    value               = "${var.hostname_prefix}-#instanceid.${var.vpc_name}.testing@${var.internal_zone_id}"
    propagate_at_launch = true
  }
}

module "autoscale_dns" {
  source = "meltwater/asg-dns-handler/aws"
  version = "x.y.z"

  autoscale_handler_unique_identifier = "my_asg_handler"
  autoscale_route53zone_arn           = var.internal_zone_id
  vpc_name                            = var.vpc_name
}
```

## Difference between Lifecycle action

Lifecycle_hook can have `CONTINUE` or `ABANDON` as default_result. By setting
default_result to `ABANDON` will terminate the instance if the lambda function
fails to update the DNS record as required. `Complete_lifecycle_action` in lambda
function returns `LifecycleActionResult` as `CONTINUE` on success to Lifecycle_hook.
But if lambda function fails, Lifecycle_hook doesn't get any response from
`Complete_lifecycle_action` which results in timeout and terminates the instance.

At the conclusion of a lifecycle hook, the result is either ABANDON or CONTINUE.
If the instance is launching, CONTINUE indicates that your actions were successful,
and that the instance can be put into service. Otherwise, ABANDON indicates that
your custom actions were unsuccessful, and that the instance can be terminated.

If the instance is terminating, both ABANDON and CONTINUE allow the instance to
terminate. However, ABANDON stops any remaining actions, such as other lifecycle
hooks, while CONTINUE allows any other lifecycle hooks to complete.

## License and Copyright

This project was built at Meltwater. It is licensed under the [Apache License 2.0](LICENSE).
