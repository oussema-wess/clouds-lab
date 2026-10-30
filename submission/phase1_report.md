# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->253.6<!--/answer--> | <!--answer:vm_warm_median_ms-->212.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2053.1<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->634.1<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1425.4<!--/answer--> | <!--answer:function_warm_median_ms-->641.8<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
[OK  ] vm        cold=  253.6ms  warm_median=  212.4ms
            #############
[OK  ] cloudrun  cold= 2053.1ms  warm_median=  634.1ms
            ########################################
[OK  ] function  cold= 1425.4ms  warm_median=  641.8ms
            ########################################
```
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The VM doesn't scale at all. Nothing gets created when traffic goes up: it's one e2-micro running one container, and gunicorn (1 worker, 8 threads) can only serve around 8 requests at a time, so the rest just wait in line. The limit is the machine I picked, and the only way to handle more would be to resize it or add VMs myself.

Cloud Run reacts on its own: when more requests come in, it starts new container instances (starting from zero, since I set min-instances to 0), and each one can take up to 80 requests at once. The ceiling is the --max-instances=3 flag I passed, so about 240 concurrent requests before people start getting errors.

The Cloud Function also adds instances automatically, but each instance only handles one request at a time, so ten simultaneous requests mean ten instances. Its limit is the max-instances setting, which I left at the default.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM took by far the most work. I had to build the network myself (a custom VPC, a subnet and two firewall rules tied to a network tag), create the VM, give its service account permission to read from Artifact Registry, SSH in, install Docker, log it into the registry and finally start the container. And it doesn't stop there: if I kept it, I'd also have to look after the OS, Docker and the container over time.

Cloud Run was a single command once the image existed. I still had to write the Dockerfile and build the image, but Google took care of everything else: servers, HTTPS, scaling and restarts.

The Cloud Function was the easiest: one command pointing at my source folder, with no Dockerfile and no image to build. Google packaged it for me. The downside is that I have less control over how it runs.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✓
SERVICE: echo
REGION: us-central1
URL: https://echo-128061216571.us-central1.run.app
LAST DEPLOYED BY: service-128061216571@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-28T23:16:23.761164Z

✓
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-128061216571.us-central1.run.app
LAST DEPLOYED BY: oussema.oueslati@eurecom.fr
LAST DEPLOYED AT: 2026-10-28T22:56:38.609342Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows two Cloud Run services, not one: echo-bot, which I deployed myself, and echo, which was deployed by the Cloud Functions robot account (gcf-admin-robot). So my "serverless function" is actually running as a Cloud Run service. When I ran gcloud functions deploy, Google uploaded my source, built a container image from it with Cloud Build (it ended up in the gcf-artifacts repository) and deployed that image to Cloud Run.

That explains why the two cold starts are so close: in both cases Cloud Run has to find a machine, pull an image and start the server from nothing. The VM has no cold start at all because its container never stops.

The real difference is in the packaging and the defaults. For echo-bot I wrote the Dockerfile and chose the base image and the server (gunicorn), while Google built the function's image for me. The function also runs with different settings: one request per instance, a small CPU share (about 0.17 vCPU), 256 MB of memory and a 60 second timeout, whereas my Cloud Run service accepts up to 80 requests per instance.
<!--/answer-->
