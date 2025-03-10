# ASG DNS handler | [![Build Status](https://cloud.drone.io/api/badges/meltwater/terraform-aws-asg-dns-handler/status.svg)](https://cloud.drone.io/meltwater/terraform-aws-asg-dns-handler)

This Terraform module sets up everything necessary for dynamically setting hostnames following a certain pattern on instances spawned by AWS Auto Scaling Groups (ASGs).

Learn more about our motivation to build this module in our blog post [Dynamic Route53 records for AWS Auto Scaling Groups with Terraform](https://underthehood.meltwater.com/blog/2020/02/07/dynamic-route53-records-for-aws-auto-scaling-groups-with-terraform/).

## Maintainers

This repository and the module it houses are maintained Foundation Missions A-Team.  Should you encounter issues or require changes to code maintained in this repository, please reachout through an issue that is part of this project.

## Requirements

- [Terraform](https://www.terraform.io/downloads.html) 0.12+ and [Terraform AWS provider](https://github.com/terraform-providers/terraform-provider-aws) 2.0+ use v2+ releases
- [Terraform](https://www.terraform.io/downloads.html) 0.11 and below, [Terraform AWS provider](https://github.com/terraform-providers/terraform-provider-aws) 2.0.14 and below use v1.x releases

## How do I use it?

Create an ASG and set the `asg:hostname_pattern` tag for example like this:

```
asg-test-#instanceid.asg-handler-vpc.testing@Z3QP9GZSRL8IVA
```

`#instanceid`  is converted by a Lambda function within this module to the actual AWS instance_id that corresponds to the launched instance.  The `@` symbol is used to split the FQDN from the Route 53 zone_id.

This could be interpolated in Terraform like this:

```hcl
tag {
  key                 = "asg:hostname_pattern"
  value               = "${var.hostname_prefix}-#instanceid.${var.vpc_name}.testing@${var.internal_zone_id}"
  propagate_at_launch = true
}
```

Once you have your ASG set up, you can just invoke this module and point to it:
```hcl
module "clever_name_autoscale_dns" {
  source  = meltwater/asg-dns-handler/aws"
  version = "~> 2.0"
  
  # use_public_ip = true
  autoscale_handler_unique_identifier = "clever_name"
  autoscale_route53zone_arn           = "ABCDEFGHIJ123"
  vpc_name                            = "my_vpc"
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
  source  = "meltwater/asg-dns-handler/aws"
  version = "2.1.7"

  autoscale_handler_unique_identifier = "my_asg_handler"
  autoscale_route53zone_arn           = var.internal_zone_id
  vpc_name                            = var.vpc_name
}
```

## Setup for multiple DNS zones

This example (CDKTF/Typescript) demonstrates how to use the asg-dns-handler for simultaneous updating of both private and public zones.

```typescript
import { AsgDnsHandler, AsgDnsHandlerConfig } from ".gen/modules/asg-dns-handler"
import {
  AutoscalingGroup,
  AutoscalingGroupConfig,
  AutoscalingGroupInitialLifecycleHook,
  AutoscalingGroupTag
} from ".gen/providers/aws/autoscaling-group"
import { LaunchTemplate } from ".gen/providers/aws/launch-template"

interface InstanceConfigDNSHandler {
  ZoneId: string
  ZoneName: string
  Prefix: string
}

interface InstanceConfig {
  vpcId: string
  vpcName: string
  subnetId: string
  name: string
  privateDNS: InstanceConfigDNSHandler
  publicDNS: InstanceConfigDNSHandler
}

let config: InstanceConfig
let launchTemplate: LaunchTemplate

const hostnamePatternPrivate = "asg:hostname_pattern:private"
const hostnamePatternPublic = "asg:hostname_pattern:public"

const asgDNSHandlerPrivate = new AsgDnsHandler(this, "asg_dns_handler_private", <AsgDnsHandlerConfig>{
  autoscaleHandlerUniqueIdentifier: `asg_handler_private`,
  autoscaleRoute53ZoneArn: config.privateDNS.ZoneId,
  vpcName: config.vpcName,
  hostnameTagName: hostnamePatternPrivate
})

const asgDNSHandlerPublic = new AsgDnsHandler(this, "asg_dns_handler_public", <AsgDnsHandlerConfig>{
  autoscaleHandlerUniqueIdentifier: `asg_handler_public`,
  autoscaleRoute53ZoneArn: config.publicDNS.ZoneId,
  vpcName: config.vpcName,
  usePublicIp: true,
  hostnameTagName: hostnamePatternPublic
})

new AutoscalingGroup(this, "aws_autoscaling_group", <AutoscalingGroupConfig>{
  name: "asg",
  vpcZoneIdentifier: [config.subnetId],
  desiredCapacity: 1,
  maxSize: 1,
  minSize: 0,
  launchTemplate: {
    id: launchTemplate.id,
    version: "$Default"
  },
  initialLifecycleHook: [
    <AutoscalingGroupInitialLifecycleHook>{
      name: "lifecycle-launching-private",
      defaultResult: "ABANDON",
      heartbeatTimeout: 60,
      lifecycleTransition: "autoscaling:EC2_INSTANCE_LAUNCHING",
      notificationTargetArn: asgDNSHandlerPrivate.autoscaleHandlingSnsTopicArnOutput,
      roleArn: asgDNSHandlerPrivate.agentLifecycleIamRoleArnOutput
    },
    <AutoscalingGroupInitialLifecycleHook>{
      name: "lifecycle-terminating-private",
      defaultResult: "ABANDON",
      heartbeatTimeout: 60,
      lifecycleTransition: "autoscaling:EC2_INSTANCE_TERMINATING",
      notificationTargetArn: asgDNSHandlerPrivate.autoscaleHandlingSnsTopicArnOutput,
      roleArn: asgDNSHandlerPrivate.agentLifecycleIamRoleArnOutput
    },
    <AutoscalingGroupInitialLifecycleHook>{
      name: "lifecycle-launching-public",
      defaultResult: "ABANDON",
      heartbeatTimeout: 60,
      lifecycleTransition: "autoscaling:EC2_INSTANCE_LAUNCHING",
      notificationTargetArn: asgDNSHandlerPublic.autoscaleHandlingSnsTopicArnOutput,
      roleArn: asgDNSHandlerPublic.agentLifecycleIamRoleArnOutput
    },
    <AutoscalingGroupInitialLifecycleHook>{
      name: "lifecycle-terminating-public",
      defaultResult: "ABANDON",
      heartbeatTimeout: 60,
      lifecycleTransition: "autoscaling:EC2_INSTANCE_TERMINATING",
      notificationTargetArn: asgDNSHandlerPublic.autoscaleHandlingSnsTopicArnOutput,
      roleArn: asgDNSHandlerPublic.agentLifecycleIamRoleArnOutput
    }
  ],
  tag: [
    <AutoscalingGroupTag>{
      key: hostnamePatternPrivate,
      value: `${config.privateDNS.Prefix}.${config.privateDNS.ZoneName}@${config.privateDNS.ZoneId}`,
      propagateAtLaunch: true
    },
    <AutoscalingGroupTag>{
      key: hostnamePatternPublic,
      value: `${config.publicDNS.Prefix}.${config.publicDNS.ZoneName}@${config.publicDNS.ZoneId}`,
      propagateAtLaunch: true
    }
  ]
})
```


## Developers Guide / Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) to understand how to submit pull requests to us, and also see our [Code of Conduct](CODE_OF_CONDUCT.md).

## Difference between Lifecycle action

Lifecycle_hook can have `CONTINUE` or `ABANDON` as default_result. By setting default_result to `ABANDON` will terminate the instance if the lambda function fails to update the DNS record as required. `Complete_lifecycle_action` in lambda function returns `LifecycleActionResult` as `CONTINUE` on success to Lifecycle_hook. But if lambda function fails, Lifecycle_hook doesn't get any response from `Complete_lifecycle_action` which results in timeout and terminates the instance.

At the conclusion of a lifecycle hook, the result is either ABANDON or CONTINUE.
If the instance is launching, CONTINUE indicates that your actions were successful, and that the instance can be put into service. Otherwise, ABANDON indicates that your custom actions were unsuccessful, and that the instance can be terminated.

If the instance is terminating, both ABANDON and CONTINUE allow the instance to terminate. However, ABANDON stops any remaining actions, such as other lifecycle hooks, while CONTINUE allows any other lifecycle hooks to complete.

## License and Copyright

This project was built at Meltwater. It is licensed under the [Apache License 2.0](LICENSE).
