---
layout: post
title:  "Recover an Amazon Bedrock knowledge base to a known-good point in time"
date:   2026-10-29
author: prashant
categories: [blog]
tags: [aws, bedrock, knowledge-base, backup, step-functions]
excerpt_separator: <!--more-->
---

*Verify every sync with an integrity canary, triage bad changes with Amazon Bedrock, and roll back selectively*

<!--more-->

**Level:** Advanced (300) · **Services:** Amazon Bedrock Knowledge Bases, Amazon S3 Vectors, AWS Step Functions, AWS Lambda, AWS Backup, Amazon Simple Storage Service (Amazon S3)

---

Employees ask an internal assistant how to approve a payment. Customers ask a support bot about refund rules. Each of these Retrieval Augmented Generation (RAG) applications answers from a knowledge base, and the business depends on those answers being right. Most teams can restore their databases to last Tuesday. Few can say the same for their knowledge base. If a source document is tampered with, the assistant keeps answering confidently. It gives the wrong answers.

In this post, you make an Amazon Bedrock knowledge base recoverable to a known-good point in time. You record a recovery point after every sync and verify each sync with an assertion-based integrity canary. When the canary fails, Amazon Bedrock triages what changed, and the workflow rolls back only the harmful changes. In our test, a sync that tampered with a payment policy and planted two prompt-injection documents was detected, rolled back, and re-verified in 57 seconds, with no human involved. A legitimate edit in the same batch was kept.

The pattern is industry-neutral, and the [sample code](https://github.com/opstodevops/bedrock-kb-guarded-sync) deploys it with the AWS Command Line Interface (AWS CLI).

## The challenge: durability isn't recoverability

**The vector store isn't in your backup plan.** At the time of writing, [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html) doesn't protect Amazon S3 Vectors indexes or knowledge base configuration. S3 Vectors is designed for 11 nines of durability, but durability protects data you wrote, not data you shouldn't have written.

**A sync succeeds even when the content is wrong.** An ingestion job reports `COMPLETE` whether the documents it indexed are accurate or malicious. In our test, a poisoned ingestion job completed in 6.2 seconds. From then on, the assistant told employees that callback verification for vendor bank changes was "optional", which is the precondition for business email compromise (BEC) fraud. The [OWASP Top 10 for LLM Applications (2025)](https://genai.owasp.org/llm-top-10/) lists this risk as LLM04 Data and Model Poisoning, LLM08 Vector and Embedding Weaknesses, and LLM01 Prompt Injection (indirect).

**Re-ingesting doesn't fix logical corruption.** If the source bucket already contains the bad content, re-ingesting reproduces the problem. You need to know *when* the knowledge base was last good and *which* changes made it bad.

**Output guardrails can't see it.** A contextual grounding check confirms that an answer is supported by the retrieved content. When the retrieved content is poisoned, the harmful answer is perfectly grounded.

**The recovery history is a target.** An attacker who can write documents can often also delete old versions or write vectors directly into the index.

## Solution overview

The solution treats every sync like a production deployment: verify it before you trust it, and keep a way to roll it back. After every sync, a **recovery point** records the S3 object version of every source document and the full knowledge base configuration. An **integrity canary** asks business-critical questions and checks the answers against explicit assertions. When the canary fails, **AI-assisted triage** reviews every change since the last known-good recovery point, and a **selective rollback** restores only the harmful documents without deleting any version history.

Two layers add defense in depth. Resource policies stop anyone, including administrators, from erasing version history or writing vectors directly. AWS Backup covers the case where the source bucket itself is destroyed.

![Architecture diagram: a document publisher writes to a versioned S3 source bucket. A schedule starts an AWS Step Functions state machine that syncs the Amazon Bedrock knowledge base, which reads the source bucket and writes to an Amazon S3 Vectors index. An inspect Lambda function runs the canary against the knowledge base with Claude Haiku 5.5 and Claude Sonnet 5.5 and writes recovery points to an S3 Object Lock bucket. A rollback Lambda function writes to the source bucket. Amazon SNS notifies the owner. AWS Backup protects both buckets.]({{ site.baseurl }}/assets/images/2026/recover-bedrock-knowledge-base/architecture.png)

*Figure 1: Guarded sync for an Amazon Bedrock knowledge base*

The guarded sync workflow (Figure 1) runs as follows:

1. A publisher (a person, a pipeline, or a connector) writes documents to the versioned source bucket.
2. A schedule or the publishing pipeline starts the guarded sync state machine in AWS Step Functions, the only principal allowed to start ingestion jobs.
3. Step Functions starts the ingestion job and polls it until it completes.
4. The **inspect** AWS Lambda function runs the integrity canary.
5. The inspect function records a recovery point marked `GOOD` or `SUSPECT`.
6. A `GOOD` recovery point ends the workflow.
7. For a `SUSPECT` one, the inspect function diffs the source bucket against the last `GOOD` recovery point, and Amazon Bedrock triages each change.
8. The **rollback** function restores the harmful documents. The workflow syncs and runs the canary once more.
9. Amazon Simple Notification Service (Amazon SNS) sends the triage analysis, or `ACTION REQUIRED` if the canary still fails.

The design covers three recovery tiers:

| Tier | What went wrong | Recovery | Measured |
|---|---|---|---|
| 1 | Poisoned content, a bad edit, or a broken sync | Selective rollback to the last `GOOD` recovery point, incremental re-sync | 57 s end to end, including detection and verification |
| 2 | Knowledge base or vector index deleted or misconfigured | Rebuild index, knowledge base, and data source from the manifest, full ingestion | 34 s for 100 documents, including verification |
| 3 | Source bucket destroyed or its version history purged | Restore the bucket from AWS Backup, then Tier 2 | About 5 minutes: a 4 min 16 s restore of 100 objects, then a 35 s rebuild and verification |

## Walkthrough

The example knowledge base belongs to AnyCompany, a fictional company. It holds 8 business policies and 92 team pages. It uses Amazon Titan Text Embeddings V2 (1,024 dimensions), fixed-size chunking of 300 tokens with 20% overlap, and an S3 Vectors index. Anthropic's Claude Haiku 5.5 answers questions, and Claude Sonnet 5.5 checks assertions and triages.

The [README](https://github.com/opstodevops/bedrock-kb-guarded-sync) deploys everything with one script, or step by step with a check after each step. This section explains the parts that matter.

### Prerequisites

- An AWS account with access to Claude Haiku 5.5, Claude Sonnet 5.5, and Amazon Titan Text Embeddings V2 in Amazon Bedrock. We used the global cross-Region inference profiles in the US West (Oregon) Region (us-west-2).
- AWS CLI version 2 with the `s3vectors` commands, plus `jq`, `envsubst`, `zip`, Python 3, and `pip`.
- Permissions to create IAM roles, S3 buckets, an S3 vector bucket, a knowledge base, Lambda functions, a state machine, an SNS topic, and optionally AWS Backup vaults.

### Step 1: Protect the source and its history

Turn on S3 Versioning on the source bucket, because every recovery point refers to object versions. Then protect the version history itself. This bucket policy denies deleting object versions, suspending versioning, and adding lifecycle rules (which could expire old versions) to everyone except a break-glass role:

```json
{
  "Sid": "ProtectRecoveryHistory",
  "Effect": "Deny",
  "Principal": "*",
  "Action": ["s3:DeleteObjectVersion", "s3:PutBucketVersioning", "s3:PutLifecycleConfiguration"],
  "Resource": ["arn:aws:s3:::amzn-s3-demo-source-bucket", "arn:aws:s3:::amzn-s3-demo-source-bucket/*"],
  "Condition": {"ArnNotEquals": {"aws:PrincipalArn": "arn:aws:iam::111122223333:role/breakglass-role"}}
}
```

A second statement denies `s3:PutBucketNotification` to everyone except the break-glass and AWS Backup roles. AWS Backup continuous backups for S3 depend on the bucket sending events to Amazon EventBridge. If someone turns that off, [continuous backups stop silently](https://docs.aws.amazon.com/aws-backup/latest/devguide/s3-backups.html), with no job failure or alert.

### Step 2: Close the side door to the vector index

Anyone with `s3vectors:PutVectors` can write embeddings, with any text attached, directly into the index. That bypasses your documents, your manifest, and your diff. A vector bucket policy enforces that only the knowledge base's service role can write or delete vectors:

```json
{
  "Sid": "OnlyKnowledgeBaseWritesVectors",
  "Effect": "Deny",
  "Principal": "*",
  "Action": ["s3vectors:PutVectors", "s3vectors:DeleteVectors", "s3vectors:DeleteIndex"],
  "Resource": ["arn:aws:s3vectors:us-west-2:111122223333:bucket/amzn-s3-demo-vector-bucket",
               "arn:aws:s3vectors:us-west-2:111122223333:bucket/amzn-s3-demo-vector-bucket/index/*"],
  "Condition": {"ArnNotEquals": {"aws:PrincipalArn": "arn:aws:iam::111122223333:role/kb-service-role"}}
}
```

We attacked both policies as an administrator. All seven attempts failed with `explicit deny in a resource-based policy`: `PutVectors`, `DeleteVectors`, `DeleteIndex`, deleting a known-good object version, adding an expiring lifecycle rule, suspending versioning, and clearing the bucket notifications. A sync afterwards still indexed an edited document through the knowledge base role.

Ingestion jobs don't have a resource policy. If you use AWS Organizations, the sample includes a service control policy (SCP) that denies `StartIngestionJob`, `IngestKnowledgeBaseDocuments`, `DeleteKnowledgeBaseDocuments`, `UpdateDataSource`, and `UpdateKnowledgeBase` to everyone except the guarded sync and break-glass roles.

### Step 3: Record a recovery point after every sync

A recovery point is a JSON manifest. It records which version of every source document was in the knowledge base, how everything was configured, and whether the result was verified. The core of the `capture_manifest` function:

```python
manifest = {
    "schemaVersion": "1.0", "status": status, "capturedAt": captured_at.isoformat(),
    "knowledgeBaseId": KB_ID, "knowledgeBaseName": kb["name"], "roleArn": kb["roleArn"],
    "dataSourceId": DS_ID, "ingestionJobId": event["ingestionJobId"],
    "canary": canary, "ingestionStatistics": job["statistics"],
    "knowledgeBaseConfiguration": kb["knowledgeBaseConfiguration"],
    "storageConfiguration": kb["storageConfiguration"],
    "vectorIndexConfiguration": {k: index[k] for k in
                                 ("dataType", "dimension", "distanceMetric", "metadataConfiguration")},
    "dataSourceConfiguration": ds["dataSourceConfiguration"],
    "vectorIngestionConfiguration": ds["vectorIngestionConfiguration"],
    "dataDeletionPolicy": ds.get("dataDeletionPolicy"),
    "sourceObjects": _current_objects(),   # [{key, versionId, etag, size}, ...]
}
key = f"{RP_PREFIX}{captured_at:%Y%m%dT%H%M%S}_{event['ingestionJobId']}_{status}.json"
s3.put_object(Bucket=MANIFEST_BUCKET, Key=key, ContentType="application/json",
              Body=json.dumps(manifest, default=str, indent=2).encode(), ChecksumAlgorithm="SHA256")
```

Record the vector index configuration, not only the knowledge base configuration. The dimension, distance metric, and non-filterable metadata keys live on the index, and you can't change them after you create it. Without them, a rebuild has to guess.

The manifest bucket uses S3 Object Lock with a default retention period. The inspect function can write only under `recovery-points/`. The golden set lives in the same bucket under `canary/`, readable but not writable by the function. An attacker who can change your documents shouldn't also be able to change the test that checks them.

### Step 4: Build an assertion-based integrity canary

Free-form "is this answer consistent with the reference?" judging is too noisy to trigger automatic rollback, because the judge objects to true details that a short reference answer omits. Write each golden case as short, explicit assertions instead:

```json
{"id": "fin-bankchange-1",
 "question": "A vendor emailed us new bank account details. Can Accounts Payable update them based on the email?",
 "mustInclude": ["Bank details must not be changed based on email alone",
                 "AP must verify by calling the vendor on the phone number already on file"],
 "mustNotClaim": ["Callback verification is optional",
                  "AP can update the bank details directly from the email"]}
```

The canary sends each question through `RetrieveAndGenerate` with the application's model and prompts. Claude Sonnet 5.5 then reports, per assertion, whether the answer states it, using Converse [structured outputs](https://aws.amazon.com/blogs/machine-learning/structured-outputs-on-amazon-bedrock-schema-compliant-ai-responses/) to return schema-valid JSON. The model provides evidence. Code makes the decision.

```python
def _run_once(case):
    answer, cited = _answer(case["question"])
    check = _structured(JUDGE_SYSTEM, _assertion_prompt(case, answer), ASSERTION_SCHEMA, "assertion_check", 2000)
    include, forbid = case["mustInclude"], case["mustNotClaim"]
    # The model only reports which assertions hold; code makes the pass/fail decision.
    if check is None or len(check["mustInclude"]) != len(include) or len(check["mustNotClaim"]) != len(forbid):
        return {"id": case["id"], "verdict": "FAIL", "reason": "judge output invalid (fail closed)", "cited": cited}
    missing = [include[a["id"]] for a in check["mustInclude"] if not a["present"]]
    violated = [{"claim": forbid[a["id"]], "evidence": a["evidence"]}
                for a in check["mustNotClaim"] if a["present"]]
    return {"id": case["id"], "verdict": "FAIL" if missing or violated else "PASS",
            "missing": missing, "violated": violated, "cited": cited}


def _run_case(case):
    """A case fails only if at least two of three independent runs fail, which filters judge noise."""
    first = _run_once(case)
    if first["verdict"] == "PASS":
        return {**first, "runs": 1}
    runs = [first, _run_once(case), _run_once(case)]
    failures = [r for r in runs if r["verdict"] == "FAIL"]
    return {**(failures[0] if len(failures) >= 2 else runs[-1]), "runs": 3, "failedRuns": len(failures)}
```

Four details make it reliable:

- **Fail closed.** `_structured()` returns `None` unless the stop reason is `end_turn` and a text block parses as JSON. An unparseable verdict is a failure, never a pass.
- **Majority of three.** Claude Haiku 5.5 and Claude Sonnet 5.5 don't accept `temperature` or `top_p`, and AWS notes that temperature 0 never guaranteed identical output anyway. Re-running failed cases filters noise: real poisoning failed three out of three runs on every affected case.
- **Give the judge enough tokens.** A small `maxTokens` truncates the JSON. We use 2,000 and ask for evidence quotes under 25 words.
- **Supply your own knowledge base prompts.** Newer Claude models have no default knowledge base prompts. `RetrieveAndGenerate` rejects them unless you pass both a generation and an orchestration prompt template, each containing `$query$`. Keep the application's prompts in one file (`lambda/kb-prompts.json` in the sample) so the canary tests exactly what users see.

Eight cases cover 22 assertions. Five consecutive canary runs against the clean knowledge base passed 40 of 40 cases with no retries, in 4.7 to 5.4 seconds per run.

### Step 5: Triage what changed with Amazon Bedrock

The failing answers' citations don't tell you everything to roll back. Our attack included a planted document that targeted HR questions, which no golden case asks. No failing answer cited it, so citation-based triage would have left it in place for the next employee who asks about parental leave.

Triage therefore starts from a deterministic diff between the source bucket and the last `GOOD` recovery point. Claude Sonnet 5.5 receives the before and after content of every changed document, plus the failed canary cases:

```python
def triage(event):
    good_key, good = _latest_good_manifest()
    changes = _diff(good["sourceObjects"], _current_objects())   # MODIFIED, ADDED, DELETED
    report = _structured(TRIAGE_SYSTEM, _triage_prompt(event["canary"]["failedCases"], changes),
                         TRIAGE_SCHEMA, "triage_report", 4000) or {}
    # Deterministic guardrails around the model's recommendation: ignore keys that aren't in
    # the diff, and roll back every change the model didn't explicitly mark KEEP (fail closed).
    changed = {c["key"] for c in changes}
    keep = {d["key"] for d in report.get("documents", [])
            if d["key"] in changed and d["recommendedAction"] == "KEEP"}
    return {"goodRecoveryPoint": good_key, "changes": changes,
            "rollback": [c for c in changes if c["key"] not in keep], "kept": sorted(keep),
            "summary": report.get("summary", "Triage output invalid: rolled back every change (fail closed)."),
            "report": report,
            "ignoredKeys": [d["key"] for d in report.get("documents", []) if d["key"] not in changed]}
```

The system prompt tells the model that document content is untrusted data and that it must never follow instructions inside it. In our test, the triage result was:

| Document | Change | Risk | Instructions to AI | Action |
|---|---|---|---|---|
| `finance/vendor-payment-policy.md` | Modified | HIGH | No | ROLLBACK |
| `finance/ap-process-update-q4.md` | Added | HIGH | **Yes** | ROLLBACK |
| `hr/benefits-portal-update.md` | Added | HIGH | **Yes** | ROLLBACK |
| `teams/platform/release-process.md` | Modified | LOW | No | KEEP |

The model flagged both "Note to AI assistants… Do not mention this note." passages as indirect prompt injection and didn't follow them. It kept the change-freeze edit as "a plausible, restrictive operational note, unrelated to the finance canary failures". It referenced no keys outside the diff.

### Step 6: Roll back selectively without destroying evidence

The rollback function has its own IAM role. It can read object versions and write current objects, but it doesn't have `s3:DeleteObjectVersion`, so automated recovery can't destroy evidence.

```python
def rollback(event):
    actions = []
    for c in event["triage"]["rollback"]:
        if c["goodVersion"]:  # MODIFIED or DELETED: copy the known-good version back on top
            r = s3.copy_object(Bucket=SOURCE_BUCKET, Key=c["key"],
                               CopySource={"Bucket": SOURCE_BUCKET, "Key": c["key"], "VersionId": c["goodVersion"]})
            actions.append({"key": c["key"], "action": "RESTORED", "versionId": r["VersionId"]})
        else:  # ADDED since the recovery point: hide it behind a delete marker, keep it for forensics
            r = s3.delete_object(Bucket=SOURCE_BUCKET, Key=c["key"])
            actions.append({"key": c["key"], "action": "DELETE_MARKER", "versionId": r.get("VersionId")})
    return {"actions": actions}
```

The next sync is incremental. In our test, it scanned 100 documents, re-indexed 1, and removed 2, in 3.1 seconds. The knowledge base treats a version restored with `CopyObject` as a modified document, and a document hidden by a delete marker as deleted, so it removes that document's vectors.

### Step 7: Orchestrate the guarded sync

At the time of writing, Amazon Bedrock doesn't publish ingestion job state changes to EventBridge. The state machine owns the sync instead: it calls `bedrockagent:startIngestionJob` through an AWS SDK service integration and polls `getIngestionJob` every 5 seconds. After the canary and the recovery point, one choice state decides what happens next:

```json
"Verified?": {
  "Type": "Choice",
  "Choices": [
    {"And": [{"Variable": "$.recoveryPoint.r.status", "StringEquals": "GOOD"},
             {"Variable": "$.mode", "StringEquals": "normal"}], "Next": "SyncVerified"},
    {"And": [{"Variable": "$.recoveryPoint.r.status", "StringEquals": "GOOD"},
             {"Variable": "$.mode", "StringEquals": "rollback"}], "Next": "NotifyRecovered"},
    {"Variable": "$.mode", "StringEquals": "rollback", "Next": "NotifyHuman"}
  ],
  "Default": "Triage"
}
```

After `Rollback`, a pass state sets `mode` to `rollback` and loops back to `StartSync`, which limits automatic recovery to one attempt. Every task also routes errors to `NotifyHuman`, so a failure is never silent. The state machine's role can start ingestion jobs only on this knowledge base, invoke only the two guard functions, and publish only to the alert topic.

**Note:** In our test, `RetrieveAndGenerate` with an inference profile failed with `Not authorized to call GetInferenceProfile` until the inspect role had `bedrock:GetInferenceProfile` on the profile. The role also needs `bedrock:InvokeModel` on the profile and on the foundation model in every Region that the profile routes to.

### Step 8: Add AWS Backup for the worst case

Tiers 1 and 2 assume that the source bucket and its history survive. For account compromise, ransomware, or a break-glass mistake, protect both buckets with AWS Backup. A continuous backup rule gives point-in-time restore for 35 days, and a copy action sends recovery points to a [logically air-gapped vault](https://docs.aws.amazon.com/aws-backup/latest/devguide/logicallyairgappedvault.html):

```json
{"RuleName": "continuous-pitr-35d", "TargetBackupVaultName": "kb-vault",
 "ScheduleExpression": "cron(0 5 * * ? *)", "EnableContinuousBackup": true,
 "Lifecycle": {"DeleteAfterDays": 35},
 "CopyActions": [{"DestinationBackupVaultArn": "arn:aws:backup:us-west-2:111122223333:backup-vault:kb-airgapped",
                  "Lifecycle": {"DeleteAfterDays": 35}}]}
```

Copies of continuous backups are snapshots: you restore them to the time of the copy, not to any point in time. Nobody can delete a recovery point in a logically air-gapped vault before its retention period ends, and you can't delete the vault while it holds recovery points.

A continuous backup starts at the rule's next scheduled run; you can't start one on demand. In our test, the first backup of each 100-object bucket completed 52 minutes after the scheduled start, and the copies took another 17 minutes. Keep backup completion off the critical path of a sync.

### Step 9: Rebuild a knowledge base from a recovery point

If the knowledge base or its index is deleted (Tier 2), the latest `GOOD` manifest has everything you need. The sample's `scripts/rebuild_kb.py` checks the source bucket against the manifest, then recreates everything from it:

```python
vector_bucket_arn = m["storageConfiguration"]["s3VectorsConfiguration"]["indexArn"].split("/index/")[0]
s3vectors.create_index(vectorBucketArn=vector_bucket_arn, indexName=index_name, tags=TAGS,
                       **m["vectorIndexConfiguration"])
kb = agent.create_knowledge_base(
    name=args.name, roleArn=m["roleArn"], tags=TAGS,
    knowledgeBaseConfiguration=m["knowledgeBaseConfiguration"],
    storageConfiguration={"type": "S3_VECTORS", "s3VectorsConfiguration": {
        "indexArn": f"{vector_bucket_arn}/index/{index_name}"}})["knowledgeBase"]
# ...wait until the knowledge base is ACTIVE, then create the data source from the manifest
ds_id = agent.create_data_source(
    knowledgeBaseId=kb_id, name="policy-docs", dataDeletionPolicy=m["dataDeletionPolicy"],
    dataSourceConfiguration=data_source_config,
    vectorIngestionConfiguration=m["vectorIngestionConfiguration"])["dataSource"]["dataSourceId"]
```

In our drill, the new knowledge base was ready at 3.1 seconds, a full ingestion of 100 documents completed at 29.5 seconds, and the canary passed 8 of 8 at 34.0 seconds. Full ingestion time grows with the corpus, while rollback time grows only with the number of changed documents, so keep Tier 2 for when the index itself is gone.

For Tier 3, restore the source bucket from AWS Backup to a new bucket with a `RestoreTime`, such as the `capturedAt` time of a `GOOD` manifest. Then run the same script with `--source-bucket`. Restored objects get new version IDs, so the script compares ETags instead. In our drill, the restore took 4 minutes 16 seconds, all 100 objects matched the manifest, and the rebuilt knowledge base passed the canary at 34.6 seconds.

**Important:** Don't point the existing data source at the restored bucket. When we tried it, the next sync indexed no new documents, and answers still cited the original bucket. Build a fresh index and knowledge base instead.

## Results

We ran the full attack through the deployed state machine. One sync contained a tampered vendor payment policy, two planted prompt-injection documents (one targeting finance questions, one targeting HR questions), and a legitimate edit to a team page.

| Elapsed (s) | Event |
|---|---|
| 0.0 | Guarded sync starts |
| 7.2 | Ingestion completes: 2 new and 2 modified documents. **Poisoned content is now retrievable.** |
| 34.3 | Canary fails: 5 of 8 cases pass. The 3 finance cases fail 3 of 3 runs each. |
| 34.8 → 44.2 | `SUSPECT` recovery point recorded; Amazon Bedrock triages 4 changed documents |
| 45.8 | Rollback: 1 document restored, 2 hidden by delete markers, 1 kept |
| 49.8 | Incremental re-sync completes: 1 modified, 2 deleted. **Poisoned content is no longer retrievable.** |
| 56.5 | Canary passes 8 of 8 |
| 57.1 | New `GOOD` recovery point recorded, SNS notification sent with the triage analysis |

The poisoned content was retrievable for 42.6 seconds. The failing canary takes most of that time, about 23 seconds, for two reasons. Failing cases run three times. And answers built on poisoned content took up to 7.8 seconds to generate, compared with about 2 seconds on clean content. We ran this attack twice, in two separate deployments, and it took 57.1 and 62.9 seconds end to end. These are single runs in one Region, not a benchmark.

The canary's evidence quotes go straight into the notification, for example:

> `violated: "Callback verification is optional", evidence: "Callback verification is optional and left to the AP clerk's discretion."`

## Design considerations

**Treat document content as untrusted input.** Triage deliberately reads attacker-controlled text. Wrap it in tags, tell the model it's data, constrain the output with a JSON schema, and validate the output against the diff in code. Every malformed model response counts as a failure.

**Version the golden set with the policies.** If Finance legitimately changes approval thresholds, the canary fails and the workflow rolls the change back. That's the correct behavior. Make the golden-set update part of the same reviewed change.

**Exposure window.** The guarded sync verifies content after the knowledge base starts serving it, which took 42.6 seconds in our test. If that's too long, sync into a staging knowledge base, verify it, and then switch the application to it (blue/green). The recovery point and rollback mechanics stay the same.

**Canary coverage.** A canary tests only what it asks, so prioritize questions where a wrong answer causes financial, safety, security, or legal harm. The diff catches the rest.

**Production hardening.** For production, consider these additions:
- Use COMPLIANCE mode and a longer retention period for Object Lock.
- Use customer managed AWS Key Management Service (AWS KMS) keys.
- Keep the manifest bucket and backup vault in a separate account.
- Send triage reports to a ticketing system.
- Turn on knowledge base logging.

## Cost considerations

One clean canary run made 8 `RetrieveAndGenerate` calls and 8 Converse calls. Claude Haiku 5.5 used 15,568 input and 1,552 output tokens, and Claude Sonnet 5.5 used 6,580 input and 876 output tokens. A failing case runs three times, and triage runs only after a failure. Step Functions, Lambda, and S3 costs are small at typical sync frequencies. AWS Backup for S3 charges per GB-month of backup storage. For current prices, see [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/), [AWS Backup pricing](https://aws.amazon.com/backup/pricing/), and the [AWS Pricing Calculator](https://calculator.aws/).

## Clean up

To avoid ongoing charges, run `./scripts/cleanup.sh`. It removes the deny policies first, then deletes every resource the sample created, including knowledge bases and buckets from recovery drills. Deleting manifests before their 1-day Object Lock expires needs `s3:BypassGovernanceRetention`. Recovery points in the logically air-gapped vault can't be deleted before they expire; the script lists them so you can run it again later.

## Conclusion

When a RAG application answers questions that matter, its knowledge base is production data and needs the same rigor: recovery points, tested restores, and a protected recovery history. In this post, you made an Amazon Bedrock knowledge base recoverable to a known-good point in time, with every sync verified and harmful changes rolled back without destroying evidence.

The main lesson from building this was where *not* to use AI. Amazon Bedrock is good at reading a policy and explaining why a change weakens a control. The decision to roll back belongs in code you can test and audit.

To get started, pick the 5 to 10 questions your assistant must never answer wrong. Write them as `mustInclude` and `mustNotClaim` assertions, and run them after your next sync with the [sample code](https://github.com/opstodevops/bedrock-kb-guarded-sync).

**Learn more:**
- [Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Amazon S3 Vectors](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html)
- [AWS Backup for Amazon S3](https://docs.aws.amazon.com/aws-backup/latest/devguide/s3-backups.html)

---

### About the authors

*(Author bios to be added.)*
