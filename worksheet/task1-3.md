# Week 6 worksheet: serving patterns
 
**Name:** Htet Aung Lin
 
**Student ID:** 01326140
 
This is **individual work**: each student completes and submits their own copy, in their own fork.
 
Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in English.
 
## Task 0: Baseline (copy the baseline results from `results/runs.csv` here)
 
| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 505 | 622 | 93.5% | 100% |
| t0-small | 40 | 85 | 72.5% | 0% |
 
**Q0.** What does the large model buy, and what does it cost?
 
> The large model is more accurate, getting 93.5% correct compared with 72.5% for the small model. However, it also takes longer and costs more because every review goes through the large model.It also reaches its limit of 20 calls per second so it cannot easily handle more requests.
 
## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here)
 
| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 34.6 | 243 | 424 | 26 | 0 | 93.7% |
| t1-after | 9.7 | 406 | 1021 | 0 | 22 | 93.8% |
 
**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?
 
> The p50 is misleading because it only measures successful requests and excludes the 26 failed requests. The errors column tells the real story, while the total run time of 34.6 seconds,compared with 10.2 seconds in the baseline,also shows that the system became much slower.
 
**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.
 
> A hang is worse because it keeps connections,memory and often workers occupied in the gateway and review service for a long time. When these limited resources become full,new reviews from other customers can also get stuck. A slow request eventually finishes but a hung request may hold resources for 10–20 seconds.
 
**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.
 
> Pending reviews should be saved and sent to a background process that retries or scores them later in bulk. This means customers do not have to wait for the model to become available and reviews are not lost. This approach is T4 – Async/Batch.
 
**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?
 
> A 0.1-second timeout is shorter than even the fastest normal large-model call (around 140 ms), so almost every review would become pending instead of receiving a label. A 10-second timeout is too long and exceeds QR2's requirement that the review service waits at most 1 second, allowing hung requests to occupy resources for much longer.
 
## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here)
 
| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 13 | 517 | 91.5% | 38% |
| t2-0.7 | 0.7 | 356 | 585 | 93.0% | 53% |
| t2-0.8 | 0.8 | 484 | 701 | 93.5% | 64% |
| t2-0.9 | 0.9 | 468 | 596 | 93.5% | 77% |
 
**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.
 
> I would choose a threshold of 0.6 because it achieves 91.5% accuracy, which meets QR3's requirement of at least 90% while using only 38% large-model calls. However, I give up some accuracy and safety margin compared with a threshold of 0.8 which achieves 93.5%.
 
**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?
 
> The cascade helps p50 because more than half of the reviews are answered by the fast small model at a threshold of 0.6, giving a p50 of only 13 ms. However, p95 is still slow because every threshold sends more than 5% of reviews to the large model so the slow large-model requests are still included in the slowest 5%.
 
**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.
 
> For example, a customer writes, "The food was not bad." The small model may recognise the word "bad" and confidently classify the review as negative because it cannot understand the negation "not". The cascade trusts its prediction without checking with the large model, so the restaurant owner sees a happy review counted as negative in their review counts.
 
## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here)
 
| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 13 | 549 | 93.0% | 38% | 35% |
 
**Q3.1** Which reviews benefit from the cache, and which never will?
 
> Short, common reviews such as "Delicious!" and "Too salty" benefit most because they are more likely to appear repeatedly. The cache ignores differences in uppercase/lowercase letters and extra spaces, but punctuation still matters. Long, personal reviews rarely repeat, so they are unlikely to get cache hits.
 
**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?
 
> The old cache should be cleared or invalidated when a new model is deployed because it may contain outdated predictions. I would include the model version in the cache key so that predictions from the new model are stored separately. After clearing the cache, the hit rate temporarily drops to 0%, so more reviews go to the large model until the cache fills up again.
 
## Debrief
 
**Q4.** Name one thing you learned from these measurements that the lecture did not cover.
 
> I learned that a model taking around 200 ms does not mean the whole request will take only 200 ms. In my baseline experiment, the large model had a p50 latency of 505 ms because requests also had to wait in a queue. This showed me that queueing can have a big impact on system performance.
 