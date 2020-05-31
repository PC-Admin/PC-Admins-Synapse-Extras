# PC-Admins-Synapse-Extras

Extra steps to configure a Synapse homeserver.

***
## Licensing

This work is licensed under Creative Commons Attribution Share Alike 4.0, for more information on this license see here: https://creativecommons.org/licenses/by-sa/4.0/

***
## ReCaptcha

It's easy to write a script that repeatedly creates a user account at a homeserver when public registration is enabled. To avoid spam and non-human congestion set up ReCaptcha.

First make a Google account, then setup a reCAPTCHAv2 for your website: https://www.google.com/recaptcha/admin/create

After you have created it copy the public and private keys into your homeserver.yaml:

```
# This homeserver's ReCAPTCHA public key.
#
recaptcha_public_key: "your-recaptcha-public-key"

# This homeserver's ReCAPTCHA private key.
#
recaptcha_private_key: "your-recaptcha-private-key"

enable_registration_captcha: true
```

Restart Synapse:
```
$ sudo systemctl restart matrix-synapse
```

***
## Email

By default you cannot register an email to your synapse server, you need to set up a SES mail provider, i like mailgun.com.

First create an account at your SES provider: https://signup.mailgun.com/new/signup

Follow the prompts to register your server address.
Updated DNS records 2xTXT, 2xMX, CNAME as instructed.

Edit homeserver.yaml like so:
```
email:
   enable_notifs: true
   smtp_host: "smtp.mailgun.org"
   smtp_port: 587
   smtp_user: "postmaster@example.org"
   smtp_pass: "very-long-string"
   require_transport_security: false

   notif_from: "Your Friendly %(app)s homeserver <noreply@example>"
```

Restart Synapse:
```
$ sudo systemctl restart matrix-synapse
```

Test by linking an email with one of your accounts and doing a password reset through it.

***
## Metrics

Synapse metrics will give you a lot of insight into your server.

Edit homeserver.yaml to setup a metrics listener:
```
listeners:

# Metrics Listener
  - type: metrics
    port: 9000
    bind_addresses:
      - '0.0.0.0'

## Metrics ###

# Enable collection and rendering of performance metrics
#
enable_metrics: true
```

Restart Synapse:
```
$ sudo systemctl restart matrix-synapse
```

You should now be able to view metrics at that port.

***
## Monitoring with Prometheus and Graphana

The metrix stats are pretty hard to read, setting up Prometheus and Graphana will give us cool graphs of Synapses' performance. I prefer to do this on a seperate VM.

Install Prometheus:
https://prometheus.io/download/
```
$ wget https://github.com/prometheus/prometheus/releases/download/v2.18.1/prometheus-2.18.1.linux-amd64.tar.gz
$ tar -xf prometheus-2.18.1.linux-amd64.tar.gz
```

Copy 'synapse-v2.rules' file from: https://github.com/matrix-org/synapse/tree/master/contrib/prometheus
```
$ wget https://raw.githubusercontent.com/matrix-org/synapse/master/contrib/prometheus/synapse-v2.rules
```

Edit prometheus configuration and add Synapse metrics:
```
$ nano ./prometheus-2.18.1.linux-amd64/prometheus.yml
```
Edit so:
```
rule_files:
  - "/path/to/synapse-v2.rules"

---

  - job_name: "synapse"
    metrics_path: "/_synapse/metrics"
    # when endpoint uses https:
    #scheme: https

    # Point this to the server you opened the metrics port on:
    static_configs:
    - targets: ['192.168.1.111:9000']
```

To run Prometheus:
```
$ ./prometheus --config.file="prometheus.yml" --web.listen-address="0.0.0.0:9090" &
```

Now check out 9090 and see if you get synapse metrics in the dropdown.

Install Graphana: https://grafana.com/docs/grafana/latest/installation/debian/

Looking at the config:
```
$ sudo nano /etc/grafana/grafana.ini
```

Now start daemon:
```
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server
```

Graphana might take a few minutes to setup, inspect port 3000 and set up a web account for it.

In the web menu import synapse.json: https://github.com/matrix-org/synapse/tree/master/contrib/grafana

You should now see a lot of performance statistics and graphics in Graphana.

***
## Create your first Synapse worker

Synapse workers outsource jobs from the main Synapse process, a 'federation_sender' is a good first worker to setup as it's simple and usually gives you an impressive speedup. You can read about other workers here: https://github.com/matrix-org/synapse/blob/develop/docs/workers.md

In this example we are going to setup workers with systemd: https://github.com/matrix-org/synapse/tree/develop/docs/systemd-with-workers

1) Configure worker files as specified.
```
$ sudo mkdir /etc/matrix-synapse/workers
$ sudo chown matrix-synapse:nogroup /etc/matrix-synapse/workers
$ sudo touch /etc/matrix-synapse/workers/federation_sender_log_config.yaml
$ sudo touch /etc/matrix-synapse/workers/federation_sender.yaml
$ sudo chown matrix-synapse:nogroup /etc/matrix-synapse/workers/federation_sender.yaml
$ sudo nano /etc/matrix-synapse/workers/federation_sender.yaml
```
Add:
```
worker_app: synapse.app.federation_sender

# The replication listener on the synapse to talk to.
worker_replication_host: 127.0.0.1
worker_replication_port: 9092
#worker_replication_http_port: 9093

worker_pid_file: /etc/matrix-synapse/workers/federation_sender.pid
worker_log_config: /etc/matrix-synapse/workers/federation_sender_log_config.yaml
```

For steps 2-7 please follow: https://github.com/matrix-org/synapse/tree/develop/docs/systemd-with-workers#set-up

Edit homeserver.yaml to enable replication listeners and to stop federation from the main Synapse process:
```
$ sudo nano /etc/matrix-synapse/homeserver.yaml
```
Edit in these lines:
```
listeners:

# Replication Endpoints
  # The TCP replication port
  - port: 9092
    bind_address: '127.0.0.1'
    type: replication
  # The HTTP replication port
  - port: 9093
    bind_address: '127.0.0.1'
    type: http
    resources:
    - names: [replication]

send_federation: False
```

Restart services and check their output:
```
$ sudo systemctl restart matrix-synapse.target
$ sudo journalctl -e -u matrix-synapse
$ sudo journalctl -e -u matrix-synapse-worker@federation_sender.service
```

Be aware you'll be using new commands to stop/start these services: https://github.com/matrix-org/synapse/tree/develop/docs/systemd-with-workers#usage

Congradulations you have your first Synapse worker configured!

***
## Moderate the service with the admin APIs

You'll want to set a user account to 'server admin' so you can use their "access token" to have a play with the APIs: https://github.com/matrix-org/synapse/blob/master/docs/admin_api/user_admin_api.rst

Here is a python script I made to simplify the usage of this API: https://github.com/PC-Admin/PC-Admins-Synapse-Moderation-Tool

***
## Backup the service.

You'll need to backup the following:
- Your Synapse database. https://www.postgresql.org/docs/9.1/backup-dump.html
- Your Synapse media location, usually it's: /var/lib/matrix-synapse/media/*
- Yout Synapse configuration files: /etc/matrix-synapse/*

These are the same files/directories to copy when migrating your service as well.

***
## Tune Postgresql

Postgres performance can be analysed with Graphana, here are a few settings that should speed things up:

```
# - Memory -

shared_buffers = 1024MB
work_mem = 16MB
effective_cache_size = 4GB

synchronous_commit = off
```

***
## Adjust other settings

`What settings would we use for synapse/nginx/postgresql/coturn/jitsi if we had 1000 users?`

