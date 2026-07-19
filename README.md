# 📊 Internee.pk — Cloud Monitoring & Log Analysis (AWS CloudWatch)

Author: Umair Sajid
Environment: AWS (EC2 + Application Load Balancer)
Service: Amazon CloudWatch

---

## 🎯 1. Objective
Monitor the Internee.pk system for performance and security using native AWS CloudWatch capabilities — metrics, dashboards, alarms, log retention, and encryption — with no third-party tooling.

---

## 🏗️ 2. Architecture

```
🌐 Internet
   │
   ▼
⚖️ Application Load Balancer (ALB) → TargetResponseTime, HTTP 5XX/4XX counts
   │
   ▼
🖥️ EC2 Instance(s) — Internee.pk App
   ├─ AWS/EC2 namespace (built-in): CPUUtilization, NetworkIn/Out, StatusCheckFailed
   ├─ 🧩 CloudWatch Agent → CWAgent namespace: Memory %, Disk %
   └─ 📄 CloudWatch Agent → CloudWatch Logs: /internee/app, /internee/nginx-error (🔐 KMS-encrypted, 30-day retention)

🚨 Alarms → 📢 SNS Topic (encrypted) → 📧 Email Notification
📊 Dashboard → single-pane view of everything above
```

⚠️ Note: EC2's default metrics do not include Memory or Disk usage — these require installing the CloudWatch Agent, which publishes to a custom CWAgent namespace. This is a genuine AWS limitation, not an oversight.

---

## ⚙️ 3. Configuration

| 📁 File | 🔧 Purpose |
|---|---|
| iam-least-privilege-policy.json | Scoped IAM role — cloudwatch:PutMetricData limited to CWAgent namespace, logs:* limited to /internee/*, plus SSM + KMS access. No unnecessary wildcards. |
| cloudwatch-agent-config.json | Enables mem_used_percent, disk_used_percent, disk I/O, TCP stats, and app/nginx log shipping. |
| monitoring-stack.yaml | CloudFormation template: KMS key, encrypted log groups, SNS topic + email subscription, 4 alarms, and the dashboard. |

Deploy the agent:
```bash
sudo yum install -y amazon-cloudwatch-agent
sudo aws ssm put-parameter --name "AmazonCloudWatch-internee-agent-config" \
  --type String --value file://cloudwatch-agent-config.json
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-internee-agent-config -s
```

Deploy the stack:
```bash
aws cloudformation deploy --template-file monitoring-stack.yaml \
  --stack-name internee-monitoring \
  --parameter-overrides EC2InstanceId=i-xxxx ALBFullName=app/internee-alb/xxxx \
  TargetGroupFullName=targetgroup/internee-tg/xxxx AlertEmail=ops@internee.pk \
  --capabilities CAPABILITY_IAM
```

📸 [Screenshot Placeholder 1 — CloudFormation stack CREATE_COMPLETE]

---

## 📊 4. Dashboard — Internee-pk-Monitoring

| 🧩 Widget | 📈 Metric | 💡 Why It Matters |
|---|---|---|
| CPU Utilization | AWS/EC2 CPUUtilization | Flags compute saturation |
| Memory Used % | CWAgent mem_used_percent | No native EC2 metric; catches OOM risk |
| Disk Used % | CWAgent disk_used_percent | Prevents full-volume outages |
| Network In/Out | NetworkIn/NetworkOut | Traffic baseline & anomaly detection |
| ALB Response Time | TargetResponseTime | Direct proxy for user experience |
| HTTP Error Rates | 5XX_Count, 4XX_Count | Server vs. client-side failure signal |
| Alarm Summary | Alarm widget | At-a-glance health status |

📸 [Screenshot Placeholder 2 — Full dashboard with all widgets populated]

---

## 🚨 5. Alarms

| ⚠️ Alarm | 📏 Metric | 🎚️ Threshold | ⏱️ Evaluation |
|---|---|---|---|
| internee-high-cpu | CPUUtilization | > 80% | 3 × 5-min |
| internee-high-response-time | TargetResponseTime | > 1s | 3 × 5-min |
| internee-high-5xx-error-rate | 5XX Count | > 5 | 1 × 5-min |
| internee-low-disk-space | disk_used_percent | > 85% | 2 × 5-min |

📢 All alarms notify a single encrypted SNS topic, with OKActions configured so recovery is reported too.

📸 [Screenshot Placeholder 3 — All 4 alarms showing OK state]

---

## 🔐 6. Security & Best Practices
- ✅ Least-privilege IAM (no broad wildcards)
- ✅ KMS-encrypted logs & SNS topic
- ✅ Explicit 30-day log retention
- ✅ Deliberate TreatMissingData per alarm (agent failure = flagged, not ignored)

---

## ✅ 7. Validation

1. 🧪 Native AWS test:
```bash
aws cloudwatch set-alarm-state --alarm-name internee-high-cpu \
  --state-value ALARM --state-reason "Manual validation test"
```
2. 🔥 Realistic CPU load:
```bash
stress --cpu 4 --timeout 900
```
3. 🌐 Response-time/error simulation via hey/ab against a test endpoint.
4. 💾 Disk space test:
```bash
fallocate -l 5G /tmp/fillfile && rm /tmp/fillfile
```

📸 [Screenshot Placeholder 4 — Alarm transition OK → ALARM → OK + SNS email]

---

## 🏁 8. Conclusion
Internee.pk now has full-stack CloudWatch observability — CPU/network natively, memory/disk via the CloudWatch Agent, and latency/errors via the ALB. Four alarms cover all required failure conditions, route through an encrypted SNS pipeline, and were validated using both AWS-native testing and real load simulation. Logs are encrypted, retained for 30 days, and access is scoped to least privilege throughout.

---

📝 Prepared by Umair Sajid🚀
