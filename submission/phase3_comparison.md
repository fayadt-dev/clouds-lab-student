# Phase 3 — Comparison Writeup

---

## 1. Facts from your runs

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->55.0<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.024<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.2<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->8<!--/answer--> |

<!--answer:timing_explanation-->
Local (0.024 s) is fastest, then Spark (4.2 s), then the cloud job (55.0 s). Local and cloud do the same word count, so they are the fair comparison. The cloud job is slow because of overhead: starting functions, reading and writing files in Cloud Storage, and Workflows calling each task. Spark does more work (TF-IDF) on one machine and still takes only 4.2 s, mostly startup. So the cloud time is overhead, not computing.
<!--/answer-->

<!--answer:counts_interpretation-->
4 stages means 3 shuffles. They come from the `reduceByKey` in `term_freq`, the `reduceByKey` in `doc_freq`, and the `join`. In the chaos run, 8 attempts failed out of 24 (33%). That matches `--chaos 0.3`.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

<!--answer:transformations_vs_actions-->
Transformations: `parallelize`, `flatMap`, `map`, `reduceByKey`, `join`. Action: `collect`. `build_tfidf_rdd` returned at once: it only built the plan, nothing ran. `collect` started the real job: 4 stages, 12 tasks, 4.2 s.
<!--/answer-->

<!--answer:lineage_recovery-->
The first `reduceByKey` ends at the shuffle `ShuffledRDD[3]`, after stage 0. If one of its output partitions is lost, Spark only recomputes that partition by reading stage 0's shuffle files again. Everything else is reused. Recovery starts at that shuffle boundary.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

- **JobTracker**: <!--answer:role_jobtracker-->Cloud Workflows (`mr-wordcount`, `workflow.yaml`)<!--/answer-->
- **Workers**: <!--answer:role_workers-->The Cloud Run function `mr-worker` (`main.py`)<!--/answer-->
- **HDFS**: <!--answer:role_hdfs-->The Cloud Storage bucket `project-33260770-cf41-4259-a62-eurecomgpt`<!--/answer-->
- **Partitioner**: <!--answer:role_partitioner-->`partition()` in `mapreduce.py` (`crc32(word) % num_reducers`)<!--/answer-->

<!--answer:barrier_location-->
The `parallel` step `map_phase`. It only finishes when all map tasks are done, so `reduce_phase` cannot start before that.
<!--/answer-->

<!--answer:idempotence-->
A map task always writes the same files with the same names, and overwrites them. Running it twice gives the same result, so retrying is safe. If it appended instead, a retried task would add its words twice, the counts would be wrong, and the result would not match the local one.
<!--/answer-->

<!--answer:chaos_observations-->
I saw failed tasks being retried in the execution view. In Logs Explorer, 8 calls to `mr-worker` returned 503 (simulated crashes). The job still succeeded in 55.0 s (45.3 s without chaos), with 2221 words and `identical: True`.
<!--/answer-->
