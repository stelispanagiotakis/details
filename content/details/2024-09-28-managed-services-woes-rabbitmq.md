---
title: "Managed Services Woes, RabbitMQ Edition"
date: "2024-09-28"
---

I love managed services. You need a message broker, I'll have one ready for you in 15'.  
We trade some money for ease of use and focus on the core deliverable, great.

But then we need a simple change. Make the message queue's endpoint public.  
If you're using AWS MQ RabbitMQ and had created a private instance, then tried to flick a switch to make it public...  
Let's start a support group, I'll bring the cookies and frustration :-P

`Private` or `Public` is a decision you need to make on creation and be happy with it forever on.  
So what do we do if we now need it to be public and would really really prefer not to destroy and re-create?  
After all, this thing has data in it and it's not trivial, if at all possible, to backup and restore existing messages, see [here](https://www.rabbitmq.com/docs/backup).

Well, [this](https://aws.amazon.com/blogs/compute/creating-static-custom-domain-endpoints-with-amazon-mq-for-rabbitmq/) is an option. Create a network load balancer exposing the RabbitMQ endpoints.  
Added benefits: You get to NOT expose the management interface and on the service endpoint you can do IP Filtering.  
But this is ClickOps, how do I turn it IaC? Here you go!

> **Caveat**: when creating the `data.aws_network_interface.rabbit-enis` resource and using it in the dependent resource to create the target group attachements, terraform complains that it cannot know in advance what this will look like after querying AWS for its content. So this needs to be applied first with the attachements commented out. Then you can create its dependent resources and you're golden. Not sure if this is a limitation of terraform, the older terraform version I was using or of me. I was going to get the task finished and then look into this. but... you know, other things happened...

```bash
resource "aws_mq_broker" "mq" {
  broker_name                = "my-mq-cluster"
  publicly_accessible        = "false"
  engine_type                = "RabbitMQ"
  engine_version             = "3.X.Y"
  auto_minor_version_upgrade = "true"
  host_instance_type         = "mq.m5.large"
  storage_type               = "ebs"
  deployment_mode            = "CLUSTER_MULTI_AZ"
  apply_immediately          = "false"
  subnet_ids                 = [aws_subnet.private-subnet-1.id, aws_subnet.private-subnet-2.id, aws_subnet.private-subnet-3.id]
  security_groups            = [aws_security_group.rabbitmq.id] #allows access from VPC to ports 443 and 5671 of broker

  authentication_strategy = "simple"
  user {
    username = var.mq_user
    password = var.mq_password
  }
  encryption_options {
    use_aws_owned_key = "true" # not recommended, preferable to use CMK. Checkov will complain
  }
  logs {
    general = true
  }
  maintenance_window_start_time {
    day_of_week = "SUNDAY"
    time_of_day = "06:00"
    time_zone   = "UTC"
  }
}

output "mq_endpoints" {
  value = aws_mq_broker.mq.instances.0.endpoints.0
}

output "mq_console_url" {
  value = aws_mq_broker.mq.instances.0.console_url
}

resource "aws_lb_target_group" "mq-tg" {
  name            = "mq-tg"
  port            = 5671
  protocol        = "TLS"
  target_type     = "ip"
  ip_address_type = "ipv4"
  vpc_id          = aws_vpc.vpc.id
  health_check {
    enabled             = true
    protocol            = "HTTPS"
    port                = "443"
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }
}

data "aws_vpc_endpoint" "rabbitmq" {
  vpc_id = aws_vpc.vpc.id
  filter {
    name   = "tag-value"
    values = [aws_mq_broker.mq.id]
  }
}

data "aws_network_interface" "rabbit-enis" {
  for_each = data.aws_vpc_endpoint.rabbitmq.network_interface_ids
  id       = each.key
}

resource "aws_lb_target_group_attachment" "mq-ip-targets" {
  for_each          = data.aws_vpc_endpoint.rabbitmq.network_interface_ids
  target_group_arn  = aws_lb_target_group.mq-tg.arn
  target_id         = data.aws_network_interface.rabbit-enis[each.key].private_ip
  port              = 5671
  availability_zone = data.aws_network_interface.rabbit-enis[each.key].availability_zone
}

resource "aws_lb" "mq-nlb" {
  name               = "mq-nlb"
  internal           = false
  load_balancer_type = "network"
  subnets            = [aws_subnet.public-subnet-1.id, aws_subnet.public-subnet-2.id, aws_subnet.public-subnet-3.id]
  enable_deletion_protection = false
}

resource "aws_lb_listener" "mq-listener" {
  load_balancer_arn = aws_lb.mq-nlb.arn
  port              = "5671"
  protocol          = "TLS"
  certificate_arn   = aws_acm_certificate.my-domain-tld-cert.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.mq-tg.arn
  }
}

resource "aws_route53_record" "mq-record" {
  zone_id = aws_route53_zone.my-domain-zone.zone_id
  name    = "mq.domain.tld"
  type    = "A"

  alias {
    name                   = aws_lb.mq-nlb.dns_name
    zone_id                = aws_lb.mq-nlb.zone_id
    evaluate_target_health = true
  }
}
```
