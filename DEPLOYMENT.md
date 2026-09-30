# Deployment Evidence

Performed on September 30, 2026, in AWS Academy Learner Lab, region `us-east-1`.
Evidence below is from actual commands on the student's Windows computer and
an SSM session on the scenario-2 instance. Local warm-up logs are in `evidence/`.
The cloud stacks have been deleted; these historical URLs are not live demos.

## 1. Deployed URL and instance id

Each row came from `aws cloudformation describe-stacks --stack-name lab04-service
--query "Stacks[0].Outputs" --output json` after the corresponding create waiter.

| Deployment | InstanceId | ServiceUrl |
| --- | --- | --- |
| First healthy (`infra/params-healthy.json`) | `i-00f5594adc4c90389` | `http://ec2-54-157-12-39.compute-1.amazonaws.com:8080` |
| Scenario 2 (`infra/params-scenario2.json`) | `i-0821106e9863c5db5` | `http://ec2-54-144-93-248.compute-1.amazonaws.com:8080` |
| Healthy replacement (`infra/params-healthy.json`) | `i-0374dee97bb44b403` | `http://ec2-100-53-182-139.compute-1.amazonaws.com:8080` |

The template was `infra/template.yaml`; each creation used stack name
`lab04-service`. The first healthy stack was deleted and its delete waiter
completed before scenario 2 was created; scenario 2 was likewise deleted before
the healthy replacement. The three original output JSON files and intermediate
deletion confirmations are retained in `evidence/`.

## 2. External health check

This ran from the student's computer, not inside EC2. Curl progress output is
preserved in `evidence/aws-healthy-health.txt`; the command and response were:

```text
curl.exe http://ec2-54-157-12-39.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

The template created one `t3.micro` EC2 instance using the Amazon Linux 2023 AMI
resolved by its public SSM parameter. Its security group allowed inbound TCP
8080 for the service and TCP 22 for the SSH fallback; the actual rules include
both even though an earlier comment says nothing else is open. The instance
used the lab's existing `LabInstanceProfile` and `vockey`, while `UserData`
installed/started Docker, scheduled shutdown after four hours, and launched the
course image with `-p 8080:8080` and the chosen `PORT` environment value. The
stack outputs reported its public service URL and instance ID for external
checks and remote diagnosis.

## 4. Scenario 2 diagnosis

**Persistent failing external check:** the second attempt followed an SSM
observation proving the application was already listening, rather than treating
an early boot delay as the failure. This environment returned a timeout (curl
exit 28), rather than the example handout's connection-refused exit 7.

```text
2026-09-30T06:40:24.7092064-04:00
curl.exe --connect-timeout 10 --max-time 15 http://ec2-54-144-93-248.compute-1.amazonaws.com:8080/api/health
curl: (28) Connection timed out after 10010 milliseconds
curl exit code: 28
2026-09-30T06:41:04.2778198-04:00 Recheck after observing the application listening on 9090
curl: (28) Connection timed out after 10010 milliseconds
curl exit code: 28
```

**Instance evidence:** connected using
`aws ssm start-session --target i-0821106e9863c5db5` and executed the two
commands below. The recorded output omits terminal escape sequences, echoed
editing artifacts and the session identifier; the service evidence is unchanged.

```text
SSM session to i-0821106e9863c5db5, 2026-09-30.
Commands executed: sudo docker ps; sudo docker logs lab04-service
Output lines (terminal control sequences and echoed keystrokes omitted):
CONTAINER ID   IMAGE                                     COMMAND                  CREATED         STATUS         PORTS                                       NAMES
79e7d5d798cd   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   9 seconds ago   Up 8 seconds   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service
lab04-service listening on 9090
```

**Diagnosis:** scenario 2 sets `PortOverride` to 9090, so the process listens on
container port 9090, as the log states, but Docker still forwards host port 8080
to container port 8080, as `docker ps` states. I deleted that stack and recreated
it with `infra/params-healthy.json` (empty override, hence PORT 8080), rather than
patching the running instance and letting its configuration drift from the template.

**Healthy curl after the infrastructure replacement:**

```text
2026-09-30T06:45:36.9472466-04:00
curl.exe http://ec2-100-53-182-139.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

Commands executed after the replacement health check:

```text
aws cloudformation delete-stack --stack-name lab04-service
aws cloudformation wait stack-delete-complete --stack-name lab04-service
aws cloudformation describe-stacks --stack-name lab04-service
```

The waiter completed, and the subsequent describe command produced:

```text
2026-09-30T06:46:40.3980691-04:00 Final stack deletion completed; subsequent describe-stacks:

aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```

After deletion, End Lab was selected and confirmed in AWS Academy. The browser
then displayed **AWS Status: Terminated**. No Lab 4 stack remains running.
