## .ebextensions Overview

A directory named `.ebextensions` placed at the root of your application bundle. Contains `.config` files (YAML or JSON) that customize and configure your Elastic Beanstalk environment. Files are processed in alphabetical order.

```
my-app/
├── .ebextensions/
│   ├── 01-packages.config
│   ├── 02-files.config
│   └── 03-commands.config
├── application files...
└── ...
```

---

## The Six Sections

### 1. packages

Install packages from supported package managers before application deployment:

yaml

```yaml
packages:
  yum:
    git: []
    httpd: []
  rpm:
    epel: https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
  python:
    django: []
    boto3: ["1.9.0"]
  rubygems:
    chef: ["12.0.0"]
```

**Runs:** Before application extraction — earliest stage.

---

### 2. groups

Create Linux groups on the instance:

yaml

```yaml
groups:
  groupone: {}
  grouptwo:
    gid: "45"
```

---

### 3. users

Create Linux users on the instance:

yaml

```yaml
users:
  myuser:
    groups:
      - groupone
    uid: "50"
    homeDir: "/tmp"
```

---

### 4. files

Create files on the EC2 instance — most versatile section:

yaml

```yaml
files:
  "/etc/myapp/config.json":
    mode: "000644"
    owner: root
    group: root
    content: |
      {
        "key": "value"
      }
  "/opt/elasticbeanstalk/hooks/appdeploy/post/my-script.sh":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/bin/bash
      echo "Runs after deployment"
  "/tmp/myfile.txt":
    source: https://my-bucket.s3.amazonaws.com/myfile.txt
    authentication: S3Auth
```

**Key uses:**

- Write config files to instance
- Write scripts to platform hooks directories
- Download files from S3 or URLs

**Runs:** During deployment — files written as part of setup.

---

### 5. commands

Run shell commands on the instance **before** the application is extracted:

yaml

```yaml
commands:
  01_set_timezone:
    command: "ln -sf /usr/share/zoneinfo/UTC /etc/localtime"
  02_install_thing:
    command: "pip install something"
    ignoreErrors: false
  03_conditional:
    command: "echo hello"
    test: "test ! -f /tmp/done"
    env:
      MY_VAR: "value"
    cwd: "/tmp"
```

**Key properties:**

- `command` — the shell command to run
- `ignoreErrors` — continue even if command fails
- `test` — only run command if this test command exits 0
- `env` — environment variables for the command
- `cwd` — working directory
- Commands run in alphanumeric order by key name

**Runs:** Before application is extracted — cannot reference application files.

**Does NOT support `leader_only`.**

---

### 6. container_commands

Run commands **after** the application is extracted but **before** it is deployed to its final location:

yaml

```yaml
container_commands:
  01_migrate_db:
    command: "python manage.py migrate"
    leader_only: true
  02_collectstatic:
    command: "python manage.py collectstatic --noinput"
  03_restart:
    command: "service httpd restart"
    ignoreErrors: true
```

**Key properties:**

- Same as commands PLUS `leader_only`
- `leader_only: true` — only runs on one instance (the leader) — critical for DB migrations
- Has access to application files — runs after extraction
- Environment variables from EB are available

**Runs:** After extraction, before final deployment location. Application environment variables available.

**SUPPORTS `leader_only` — commands does NOT.**

---

### 7. option_settings

Configure Elastic Beanstalk environment options:

yaml

```yaml
option_settings:
  aws:elasticbeanstalk:application:environment:
    MY_ENV_VAR: "production"
    DB_HOST: "mydb.example.com"
  aws:autoscaling:asg:
    MinSize: "2"
    MaxSize: "10"
    HealthCheckType: ELB
    HealthCheckGracePeriod: "300"
  aws:elasticbeanstalk:environment:process:default:
    HealthCheckPath: /health
  aws:elasticbeanstalk:healthreporting:system:
    SystemType: enhanced
  aws:elb:loadbalancer:
    CrossZone: true
```

**Common namespaces:**

|Namespace|Controls|
|---|---|
|`aws:autoscaling:asg`|ASG min/max, health check type|
|`aws:elasticbeanstalk:application:environment`|Environment variables|
|`aws:elasticbeanstalk:environment:process:default`|Health check path, port|
|`aws:elasticbeanstalk:healthreporting:system`|Basic vs enhanced health|
|`aws:ec2:instances`|Instance type|
|`aws:elb:loadbalancer`|Load balancer settings|

---

### 8. services

Ensure services are running and restart when files/packages change:

yaml

```yaml
services:
  sysvinit:
    httpd:
      enabled: true
      ensureRunning: true
      files:
        - "/etc/httpd/conf/httpd.conf"
      packages:
        yum:
          - httpd
```

When the listed files change or packages are updated — the service automatically restarts.

---

## Platform Hooks — The Post-Deployment Option

For running scripts **after** application is fully deployed — not an `.ebextensions` section but used with the `files` section:

yaml

```yaml
files:
  "/opt/elasticbeanstalk/hooks/appdeploy/post/99_my_script.sh":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/bin/bash
      # Runs after every deployment completes
```

**Hook directories:**

|Directory|When it runs|
|---|---|
|`hooks/preinit/`|Instance first boot only — before app deploy|
|`hooks/appdeploy/pre/`|Before each deployment|
|`hooks/appdeploy/enact/`|During each deployment|
|`hooks/appdeploy/post/`|After each deployment — fully deployed|
|`hooks/postinit/`|Instance first boot only — after app deploy|

---

## Execution Order Summary

```
Instance launch / deployment:
        ↓
1. packages installed
2. groups + users created
3. files written to instance
4. commands run (before extraction)
        ↓
5. Application extracted from bundle
        ↓
6. container_commands run (after extraction, before final deploy)
   leader_only commands run on one instance only
        ↓
7. Application deployed to final location
        ↓
8. appdeploy/post/ hooks run
        ↓
9. services started/restarted
        ↓
10. option_settings applied (environment config — ongoing)
```

---

## Critical Exam Distinctions

|Requirement|Use|
|---|---|
|Run before extraction|`commands`|
|Run after extraction, before deploy|`container_commands`|
|Run after fully deployed|`files` → `appdeploy/post/` hook|
|DB migration (run once on one instance)|`container_commands` with `leader_only: true`|
|Install OS packages|`packages`|
|Set environment variables|`option_settings`|
|Configure ASG health check|`option_settings` → `aws:autoscaling:asg`|
|Configure health check endpoint|`option_settings` → process namespace|
|Write config files to instance|`files`|
|Restart service when config changes|`services`|

---

## Common Exam Traps

|Wrong assumption|Reality|
|---|---|
|`commands` supports `leader_only`|NO — only `container_commands`|
|`container_commands` runs after full deploy|NO — before final deploy location|
|DB migrations in `commands` block|WRONG — use `container_commands` with `leader_only`|
|`commands` has access to app files|NO — runs before extraction|
|`container_commands` has access to app files|YES — runs after extraction|
|Platform hooks need separate config|NO — write them via `files` section|
|User data runs after app deployed|NO — runs at instance launch only|