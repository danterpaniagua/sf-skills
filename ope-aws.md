# Skill: AWS Operations

Investigation and operational support for AWS infrastructure used by the SmartFran platform.

## Scope

- EC2 — instance state, security groups, AMIs
- SQS — queue health, message depth, DLQ inspection
- IAM — roles, policies, access review (read-only)
- CloudWatch — metrics, alarms, log insights
- S3 — bucket listing, object inspection (no data access)
- ECS / ECR — task state, container images
- ECS Fargate — task definitions, running tasks, logs, stopped task diagnostics
- Elastic Load Balancing — ALB/NLB target group health, listener rules, access logs

## Command Constraint

> **Read commands** (`aws ... describe-*`, `aws ... list-*`, `aws ... get-*`) — safe to run, no warning needed.
> **Write/action commands** (`aws ... create-*`, `aws ... delete-*`, `aws ... put-*`, `aws ... send-*`, `aws ... start-*`, `aws ... stop-*`, `aws ... reboot-*`) — always prefix the block with `⚠️ ACTION — this command modifies AWS state.`

Set a default region when running commands:
```bash
export AWS_DEFAULT_REGION=us-east-1
```

---

## EC2 — Instance State

```bash
# List all EC2 instances with state and type
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].{ID:InstanceId, Name:Tags[?Key=='Name']|[0].Value, State:State.Name, Type:InstanceType, IP:PrivateIpAddress, PublicIP:PublicIpAddress}" \
  --output table

# Show details for a specific instance
aws ec2 describe-instances --instance-ids <instance-id> --output json

# List security groups attached to an instance
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query "Reservations[0].Instances[0].SecurityGroups" \
  --output table
```

---

## EC2 — Security Groups

```bash
# List all security groups
aws ec2 describe-security-groups \
  --query "SecurityGroups[*].{ID:GroupId, Name:GroupName, VPC:VpcId, Description:Description}" \
  --output table

# Show inbound rules for a specific security group
aws ec2 describe-security-groups \
  --group-ids <sg-id> \
  --query "SecurityGroups[0].IpPermissions" \
  --output json

# Flag open rules (source 0.0.0.0/0)
aws ec2 describe-security-groups \
  --query "SecurityGroups[*].{SG:GroupId, Name:GroupName, Rules:IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0']]}" \
  --output json
```

---

## SQS — Queue Health

```bash
# List all SQS queues
aws sqs list-queues --output json

# Get queue attributes (depth, DLQ ARN, visibility timeout)
aws sqs get-queue-attributes \
  --queue-url <queue-url> \
  --attribute-names All \
  --output json

# Key attributes to check:
#   ApproximateNumberOfMessages        — messages in flight
#   ApproximateNumberOfMessagesNotVisible — in-flight / being processed
#   RedrivePolicy                      — DLQ config (maxReceiveCount)
```

```bash
# Check DLQ depth (paste DLQ URL from RedrivePolicy above)
aws sqs get-queue-attributes \
  --queue-url <dlq-url> \
  --attribute-names ApproximateNumberOfMessages \
  --output json
```

---

## CloudWatch — Alarms and Metrics

```bash
# List alarms in ALARM state
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --query "MetricAlarms[*].{Name:AlarmName, Metric:MetricName, Namespace:Namespace, State:StateValue, Reason:StateReason}" \
  --output table

# Get metric statistics for a specific metric (last 1h, 5-min periods)
aws cloudwatch get-metric-statistics \
  --namespace AWS/SQS \
  --metric-name NumberOfMessagesSent \
  --dimensions Name=QueueName,Value=<queue-name> \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum \
  --output table
```

---

## CloudWatch Logs — Log Insights

```bash
# List log groups
aws logs describe-log-groups \
  --query "logGroups[*].{Name:logGroupName, RetentionDays:retentionInDays, SizeMB:storedBytes}" \
  --output table

# Run a Log Insights query (last 30 min)
aws logs start-query \
  --log-group-name <log-group> \
  --start-time $(date -d '30 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 50'
# Then retrieve results:
aws logs get-query-results --query-id <query-id>
```

---

## IAM — Access Review (Read-Only)

```bash
# List IAM roles
aws iam list-roles \
  --query "Roles[*].{Name:RoleName, Created:CreateDate, Path:Path}" \
  --output table

# Show attached policies for a role
aws iam list-attached-role-policies --role-name <role-name> --output table

# Show inline policies for a role
aws iam list-role-policies --role-name <role-name> --output json

# List IAM users
aws iam list-users \
  --query "Users[*].{Name:UserName, Created:CreateDate, LastUsed:PasswordLastUsed}" \
  --output table
```

---

## ECS — Service State

```bash
# List ECS clusters
aws ecs list-clusters --output json

# List services in a cluster
aws ecs list-services --cluster <cluster-name> --output json

# Describe service (desired vs running count, events)
aws ecs describe-services \
  --cluster <cluster-name> \
  --services <service-name> \
  --query "services[0].{Status:status, Desired:desiredCount, Running:runningCount, Pending:pendingCount, Events:events[:5]}" \
  --output json

# List running tasks
aws ecs list-tasks --cluster <cluster-name> --desired-status RUNNING --output json

# List stopped tasks (last failures)
aws ecs list-tasks --cluster <cluster-name> --desired-status STOPPED --output json
```

---

## ECS Fargate — Task Diagnostics

```bash
# Describe a specific task (exit codes, stop reason, container status)
aws ecs describe-tasks \
  --cluster <cluster-name> \
  --tasks <task-id> \
  --query "tasks[0].{Status:lastStatus, StopCode:stopCode, StopReason:stoppedReason, Containers:containers[*].{Name:name, Exit:exitCode, Reason:reason}}" \
  --output json

# List task definitions for a family
aws ecs list-task-definitions \
  --family-prefix <family-name> \
  --sort DESC \
  --output table

# Show full task definition (CPU, memory, image, env vars, log config)
aws ecs describe-task-definition \
  --task-definition <family-name>:<revision> \
  --output json

# Get Fargate task logs from CloudWatch (requires log group from task definition)
aws logs get-log-events \
  --log-group-name /ecs/<family-name> \
  --log-stream-name "ecs/<container-name>/<task-id>" \
  --start-from-head \
  --output json \
  --query "events[*].{Time:timestamp, Message:message}"

# Show resource utilisation for a running task
aws ecs describe-tasks \
  --cluster <cluster-name> \
  --tasks <task-id> \
  --query "tasks[0].{CPU:cpu, Memory:memory, LaunchType:launchType, PlatformVersion:platformVersion, ConnectivityAt:connectivityAt}" \
  --output json
```

---

## Load Balancer — ALB / NLB

```bash
# List all load balancers
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[*].{Name:LoadBalancerName, DNS:DNSName, Type:Type, State:State.Code, Scheme:Scheme}" \
  --output table

# List target groups for a load balancer
aws elbv2 describe-target-groups \
  --load-balancer-arn <lb-arn> \
  --query "TargetGroups[*].{Name:TargetGroupName, Protocol:Protocol, Port:Port, HealthyThreshold:HealthyThresholdCount, UnhealthyThreshold:UnhealthyThresholdCount}" \
  --output table

# Check target health (shows healthy / unhealthy / draining per target)
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn> \
  --query "TargetHealthDescriptions[*].{Target:Target.Id, Port:Target.Port, State:TargetHealth.State, Reason:TargetHealth.Reason, Description:TargetHealth.Description}" \
  --output table

# List listeners on a load balancer
aws elbv2 describe-listeners \
  --load-balancer-arn <lb-arn> \
  --query "Listeners[*].{Port:Port, Protocol:Protocol, DefaultAction:DefaultActions[0].Type, ARN:ListenerArn}" \
  --output table

# List listener rules (priority, conditions, actions)
aws elbv2 describe-rules \
  --listener-arn <listener-arn> \
  --query "Rules[*].{Priority:Priority, Conditions:Conditions, Actions:Actions[0].Type}" \
  --output json

# Show load balancer attributes (access log config, idle timeout, deletion protection)
aws elbv2 describe-load-balancer-attributes \
  --load-balancer-arn <lb-arn> \
  --output table
```

---

## Constraints

- Default to read-only commands.
- Always warn with `⚠️ ACTION` before any command that modifies AWS state (create, delete, put, send, start, stop, reboot, purge).
- Never generate commands that purge queues, terminate instances, or modify IAM policies unless explicitly requested.
- Output commands as copy-paste blocks. User runs them and pastes results back.
