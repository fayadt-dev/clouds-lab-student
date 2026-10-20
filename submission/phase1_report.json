# Phase 1 — Comparison Report

Student: fayadt-dev — project `project-33260770-cf41-4259-a62`, region `us-central1`.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only):

<!--answer:image_size_mb-->
50
<!--/answer-->

(Artifact Registry reports `SIZE: 49989214` bytes for `echo-bot:v1`, i.e. the compressed layers. The same image is 204 MB uncompressed on disk according to `docker images`. The multi-stage build on `python:3.11-slim` keeps it well under the 400 MB target.)

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

(Custom-mode VPC, subnet `10.10.0.0/24` in `us-central1`; firewall rules `allow-echo-8080` (tcp:8080) and `allow-echo-ssh` (tcp:22), both restricted to the network tag `echo-http`. The tag scopes the rules to the instances that carry it, so only `echo-vm` gets those ports opened, not every VM on the network. A custom VPC starts with deny-all ingress, so without the port-22 rule `gcloud compute ssh` would simply hang.)

---

## 2. Measurements

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->237.1<!--/answer--> | <!--answer:vm_warm_median_ms-->211.1<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1916.5<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->639.5<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1439.7<!--/answer--> | <!--answer:function_warm_median_ms-->678.6<!--/answer--> |

<!--answer:latency_plot-->
```
measure.py summary (bars = warm median)
[OK  ] vm        cold=  237.1ms  warm_median=  211.1ms
            ############
[OK  ] cloudrun  cold= 1916.5ms  warm_median=  639.5ms
            ######################################
[OK  ] function  cold= 1439.7ms  warm_median=  678.6ms
            ########################################

Warm-latency histogram (20 requests per platform, 100 ms buckets)
             100-199  200-299  600-699  700-799
vm              0       20        0        0
cloudrun        6        0       14        0
function        4        0       15        1
```
The VM is tight (210.2–215.0 ms) and its first request is only 26 ms slower than its median: there is no real cold start, because its container never stops. Cloud Run and the Function both pay a cold start (+1.28 s and +0.76 s over their warm medians), since they had scaled to zero after 15 idle minutes. Their warm latencies are bimodal, around 130 ms and 630–700 ms, with the same two clusters on both platforms. The echo itself is trivial, so this points to the shared HTTPS serving path in front of both services rather than to my code. Each cold value is a single sample, so the 480 ms gap between the two cold starts is not significant.
<!--/answer-->

---

## 3. Per-platform analysis

<!--answer:scaling_behaviour-->
**VM:** nothing is created. One shared-core e2-micro (1 GB RAM) runs one gunicorn worker with 8 threads. With 10 or 100 simultaneous requests, the extra ones queue and latency grows until they time out. The hard limit is the size of the machine I picked (`--machine-type=e2-micro`). **Cloud Run:** new container instances are started from my `echo-bot` image as load grows. Each accepts up to 80 concurrent requests (`containerConcurrency=80`), and my `--max-instances=3` flag caps it at 3 instances (about 240 requests in flight); excess requests are rejected. **Function:** it also spawns new instances, but with a concurrency of 1, so 10 requests need about 10 instances and 100 need about 100. I set no maximum (`maxInstanceCount` is empty), so the cap is the platform default and the project quota, not a flag of mine.
<!--/answer-->

<!--answer:deployment_effort-->
**VM (IaaS):** I built and configured everything myself. That meant a custom VPC and subnet, two tag-scoped firewall rules, and an IAM role (`artifactregistry.reader`) so the VM's service account could pull the image. Then I created the instance, connected over SSH, installed Docker (34 packages, about 400 MB), configured registry authentication and started the container: roughly ten manual steps. I now also own OS patching, restarts and the always-on, billed external IP. **Cloud Run:** a single `gcloud run deploy` (14 s) pointing at the image I had already built and pushed, with no network, OS or runtime to manage. **Function:** also a single command, but from source. Google built the container itself with Cloud Build (2 min 30 s), which first required granting `cloudbuild.builds.builder` to the default compute service account.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-457662934827.us-central1.run.app
LAST DEPLOYED BY: service-457662934827@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-20T13:49:59.849270Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-457662934827.us-central1.run.app
LAST DEPLOYED BY: Fayad.toukourou@eurecom.fr
LAST DEPLOYED AT: 2026-10-20T13:43:48.401356Z
```
<!--/answer-->

<!--answer:convergence_explanation-->
The list shows two Cloud Run services, not one. `echo-bot` was deployed by me, while `echo` was deployed by `gcf-admin-robot`, the Cloud Functions service agent. A gen-2 Cloud Function *is* a Cloud Run service: Cloud Functions takes my source, builds a container image with Cloud Build (stored in the `gcf-artifacts` repository) and deploys it to Cloud Run. Its `uri` is even a `*.run.app` address. Both cold starts therefore go through the same mechanism: Cloud Run schedules a fresh instance, pulls the image and boots Python. That is why 1916.5 ms and 1439.7 ms sit in the same range, while the always-on VM shows none. What genuinely differs is packaging and defaults. For Cloud Run I build and own the image (14 s deploy), whereas for the Function Google builds it from source (2 min 30 s). The Function also runs with a concurrency of 1, 0.1666 vCPU and 256 MB instead of a concurrency of 80, so it scales out to many more instances under load. Finally, it is served by the Functions Framework rather than my own gunicorn command.
<!--/answer-->
