# Troubleshooting notes

These cases came from building the project. Use the current task's logs and deployment events; old stopped-task errors can remain visible after a correction.

## `exec /usr/local/bin/gunicorn: exec format error`

The image was built for ARM64 while the original task definition selected X86_64. The image pull completed, but the process could not execute on the selected architecture.

Check the local image:

```bash
docker image inspect flask-ecs:v2 --format '{{.Os}}/{{.Architecture}}'
```

For this project, the result must be `linux/arm64`, and the task definition must use Linux/ARM64. Create a corrected task-definition revision and update the service. Alternatively, build an AMD64 image if you deliberately choose X86_64 for the task.

## Target group shows port 80 and `Request timed out`

The original task definition had `80:80` as its actual port mapping, even though its port **name** was `5000`. Gunicorn listened on port 5000, and the task security group allowed port 5000 from the ALB, so traffic sent to port 80 failed.

Check the full path:

| Component | Expected configuration |
|---|---|
| ALB listener | HTTP 80 |
| Gunicorn bind address | `0.0.0.0:5000` |
| Task definition container port | 5000 |
| ECS service load-balancer container port | 5000 |
| Registered target port | 5000 |
| Task security-group inbound rule | TCP 5000 from the ALB security group |
| Health check | HTTP `/health`, traffic port, success code 200 |

Set a descriptive port name such as `flask-http`. Create a task-definition revision with the actual port corrected, then update both the service's task revision and its load-balancer mapping. Changing the target group's default port alone does not correct a target explicitly registered on a different port.

Do not manually register task IPs as a permanent fix: ECS must keep registration correct when tasks are replaced.

## `The container flask-demo did not have a container port 80 defined`

The new revision defined port 5000, but the service still referred to port 80. Submit the new task-definition revision and `containerPort=5000` load-balancer mapping together with `aws ecs update-service`; the README contains a parameterized example.

Keep the existing target-group ARN. A new task definition does not require rebuilding the image when only ECS configuration changes.

## No healthy target after updating

1. In ECS, inspect the latest deployment, service events, and current task logs.
2. In the target group, clear any status filters and read the exact health-status details.
3. Confirm the service points to the intended task-definition revision and the current task registers on port 5000.
4. Check the ALB's actual attached security group, task ingress source, and ALB outbound access to port 5000.
5. Confirm `/health` returns 200, the ALB enables the task's Availability Zone, and the task is running.

If the container runs but health checks time out, investigate the port and network path. If image pulls or logging initialization fail, inspect execution-role permissions, endpoint security groups, private DNS, and the S3 endpoint route-table association.

## Browser cannot reach the ALB

Use `http://YOUR_ALB_DNS_NAME`, without port 5000. Verify the ALB is Active, has an HTTP 80 listener, uses public subnets with Internet Gateway routes, and permits your current public IP in its security group. HTTPS is not configured by this project.

## A pushed image does not appear in the application

An ECR push does not replace running containers. Publish a new tag, create a task-definition revision referencing it, and update the service. Verify deployment completion and the response version. Prefer unique version tags so previous images remain usable for rollback.
