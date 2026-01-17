🚀 Zero-Downtime Blue–Green Deployment with Docker & Nginx

This project demonstrates a true zero-downtime Blue–Green deployment strategy on a single EC2 instance using Docker and Nginx (open-source) — without relying on polling scripts, sleep-based health checks, or cloud load balancers.

The deployment guarantees that users never experience 5xx errors, even when the active version fails.

🎯 Problem Statement

In a traditional Docker-based deployment on a single server:

Only one version of the app runs at a time

Deployments cause downtime

Rollbacks are slow and reactive

Health-check scripts introduce delay and instability

This project solves those problems by moving failover logic into the reverse proxy layer.

🧠 Key Idea (What Makes This Different)

Zero downtime cannot be reliably achieved with reactive scripts.
It must be enforced at the proxy level, per request.

Instead of:

monitoring the app

waiting for failure

switching traffic later

We let Nginx handle failures instantly at request time.

🏗️ Architecture Overview

Blue → Stable (old) version

Green → New version (primary)

Nginx → Reverse proxy + failover controller

Both Blue and Green containers run simultaneously on the same Docker network.

Traffic Flow
Client → Nginx → Green (primary)
                    ↓
             (failure detected)
                    ↓
           Nginx retries same request → Blue


✔ Same request
✔ No delay
✔ No error shown to user

⚙️ How Zero Downtime Is Achieved

This setup uses request-level failover, not traffic switching.

Key Nginx mechanisms used:

backup upstream

Fast upstream timeouts

Immediate retry on failure

No polling or health-check loops

No container restarts during failover

📄 Final Nginx Configuration
events {}

http {

  upstream app_backend {
    server green:80 max_fails=1 fail_timeout=0;
    server blue:80 backup;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://app_backend;

      proxy_http_version 1.1;
      proxy_set_header Connection "";

      proxy_connect_timeout 1s;
      proxy_send_timeout   1s;
      proxy_read_timeout   1s;

      proxy_next_upstream_timeout 0;
      proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
      proxy_next_upstream_tries 2;
    }
  }
}

🔁 Deployment & Rollback Behavior
Normal Operation

Green serves all traffic

Blue stays idle as rollback target

Failure Scenario

First failed request on Green

Nginx retries the same request on Blue

User receives a valid response

Green is permanently avoided until manual redeploy

Recovery

Fix Green

Restart Green container

Reload Nginx

Green becomes primary again

No scripts. No delay. No downtime.

🧪 How to Test Zero Downtime
docker stop green


Expected result:

Website continues loading

No 502 / 504 errors

Page switches to Blue instantly

❌ Why Health-Check Scripts Were Removed

Earlier approaches used scripts like:

curl checks

sleep-based polling

config rewrites on failure

These are reactive and always allow a failure window.

This project removes them entirely because:

Scripts react after users are affected

Nginx can retry requests before responding

Proxy-level failover is deterministic and instant

🧩 Scope & Limitations

This guarantees zero downtime for:

Container crashes

Network timeouts

Backend 5xx errors

It does not protect against:

Logical bugs that still return 200 OK

(No system can.)

📦 Tech Stack

Docker

Docker Compose

Nginx (open source)

Ubuntu (EC2 Free Tier compatible)

🙏 Credits & Acknowledgements

This project is inspired by and builds upon the ideas from the original repository:

Original Repo:
👉 https://github.com/startbootstrap/startbootstrap-landing-page
This is the source repo used for making this project.

🏁 Conclusion

This project demonstrates how real-world zero-downtime deployments are achieved:

Not with sleep loops

Not with reactive scripts

But with fail-fast proxies and request retries

This approach mirrors how production systems using ALB, HAProxy, and Envoy work internally.
