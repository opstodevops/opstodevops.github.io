---
layout: post
title:  "Route support tickets only when the model is sure: Jev on AWS Lambda Web Functions"
date:   2026-10-29
author: prashant
categories: [blog]
tags: [aws, lambda, jev, triage, machine-learning]
excerpt_separator: <!--more-->
---

Support teams want to route customer messages automatically, but a confidently wrong route costs more than a slow one. The useful question isn't "how accurate is the classifier?" It's "which messages can I trust it with, and which ones need a person?" To answer it, you need a model whose confidence means something and a low-cost place to run the code around it.

<!--more-->

This post pairs two tools for that job. Jev, a decision model that TypeSafe released in September 2026, takes a message and typed questions with declared options and returns a choice with a probability for every option. AWS Lambda Web Functions, a Lambda form factor launched in October 2026, runs an ordinary Node.js HTTP server with no handler adapter. On 870 labelled banking support messages, Jev routed 95.6% of messages automatically at 95% accuracy or better, with no training examples, for $0.038 per 1,000 decisions.

In this post, you review the measured results and pick a confidence threshold. Then you deploy the service with the AWS Command Line Interface (AWS CLI) and call it with a signed `curl` request. The code and evaluation scripts are in the [sample repository](https://github.com/opstodevops/jev-lambda-support-triage).

## Why confidence matters more than accuracy

A triage model makes two decisions: where a message goes, and whether to act on that answer without a person. The second decision needs a number you can threshold. A model is *calibrated* when its probabilities match reality: of all the answers it gives with probability 0.9, about 90% are correct. With a calibrated model, you automate every answer above a threshold and send the rest to a human queue.

TypeSafe calls Jev a "System One" model. It answers typed questions in a single pass and returns structured answers and probabilities instead of generated text. TypeSafe states that Jev is trained for calibration. We tested that claim on our own data.

## Solution overview

Figure 1 shows the architecture.

![Architecture diagram: in one AWS Region, a client sends a SigV4-signed HTTPS request to a Lambda Web Function endpoint with IAM authentication, which forwards it to the revision running Express on Node.js 24. The revision reads the OpenRouter API key from AWS Secrets Manager through its IAM execution role, sends the message and three typed questions to Jev through the OpenRouter System One API outside AWS, receives choices and probabilities, and writes logs and metrics to Amazon CloudWatch.]({{ site.baseurl }}/assets/images/2026/route-support-tickets-jev/architecture.png)

*Figure 1: The triage service. Everything except the Jev call runs in your AWS account, in the Region you choose.*

The service is one Express app with one main route, `POST /triage`. For each message, it asks Jev three questions in a single request:

- **Queue** (a *Choice*): which of seven queues should handle the message. One of them is `other`, for messages that aren't banking support requests.
- **Urgency** (a *Score*): routine, needed today, or urgent.
- **Self-serve** (a *Noul*, Jev's yes/no type): could a help-centre article resolve this without staff acting on the account?

Jev answers each question independently, and the routing policy stays in your code. In the sample, a message with queue confidence below 0.8 goes to human review. A confident `other` gets an out-of-scope reply, and a confident message with a self-serve probability of 0.9 or more gets a help article.

A Web Function has three resources: the **Web Function** (name and configuration), immutable **revisions** of code and settings (one per deploy, so you can roll back), and **endpoints**, HTTPS domains with an authentication type that route traffic to revisions.

Web Functions fit this workload for three reasons:

1. **No adapter code.** The app is a normal Express server. The only Lambda-specific line reads the host and port from `AWS_LAMBDA_HTTP_ENDPOINT`.
2. **Waiting isn't billed as CPU.** Each request spends most of its time waiting for Jev. Web Functions bill active CPU time, and one environment serves many requests at once. In Step 6, five environments served 200 requests with 4 seconds of CPU in total.
3. **IAM authentication on the endpoint.** With `IamAuth`, Lambda rejects any request that isn't signed with AWS Signature Version 4 (SigV4) before it reaches your code.

A regular Lambda function, now called an Event Function, would also work with an adapter. It serves one request per environment and bills the wait for Jev. Choose it if you need a runtime other than Node.js, a Region without Web Functions, or event sources such as Amazon Simple Queue Service (Amazon SQS).

## Results

We evaluated on 870 messages: 770 from the Banking77 test set (10 per intent, mapped to six queues) and 100 out-of-scope questions from CLINC150 (label `other`). We wrote the intent-to-queue mapping ourselves and marked 20 of the 77 intents as ambiguous, such as fee questions about transfers. We compared Jev with keyword rules and with TF-IDF plus logistic regression, trained on 20 examples per queue or on 10,099 examples. All results are single runs on October 14, 2026, with model `typesafe/jev-1.13-20260917` called from a Web Function in Asia Pacific (Sydney) `ap-southeast-2`.

*Table 1: Results on 870 messages. Macro-F1 averages the F1 score across the seven queues. ECE (expected calibration error) is the average gap between stated probability and actual accuracy; lower is better. "Automated at ≥95% accuracy" is the largest share of messages you can automate while those messages stay at least 95% correct.*

| Method | Labelled examples | Accuracy | Macro-F1 | ECE | Automated at ≥95% accuracy | Out-of-scope caught |
|---|---|---|---|---|---|---|
| Keyword rules | 0 | 83.5% | 0.818 | 0.115 | 0% | 91% |
| TF-IDF, 20 per queue | 140 | 72.8% | 0.713 | 0.073 | 21% | 51% |
| TF-IDF, full training set | 10,099 | 87.7% | 0.852 | 0.038 | 89% | 29% |
| Jev 1.13 | 0 | **93.3%** | **0.933** | **0.033** | **96%** | **98%** |

Figure 2 shows how accuracy changes as you automate more messages, most confident first.

![Line chart of accuracy on automated messages (y-axis, 65% to 100%) against the share of messages automated (x-axis, 0% to 100%) for four methods. Jev stays near 97% until about 90% of messages are automated and ends at 93%. TF-IDF trained on 10,099 examples starts near 100%, falls below Jev at about 85%, and ends at 88%. Keyword rules and TF-IDF with 20 examples per queue stay lower throughout.]({{ site.baseurl }}/assets/images/2026/route-support-tickets-jev/accuracy-vs-coverage.png)

*Figure 2: Accuracy versus share automated. Each point is one confidence threshold. The dotted lines mark 95% and 98% accuracy.*

Three results stand out:

1. **Jev led without training data, and its probabilities were the best calibrated.** It was 5.6 points more accurate than TF-IDF trained on 10,099 examples and had the lowest ECE. It caught 98 of 100 out-of-scope messages, against 29. We measured calibration on Jev's top-option probability.
2. **A trained model learns your conventions; Jev only knows your descriptions.** On the 20 ambiguous intents, the full TF-IDF model was more accurate (91.0% against 83.5%), because it learned how we labelled borderline cases. On the rest, Jev was ahead (96.3% against 86.7%). Jev's confidence also has a ceiling: 69% of its answers had a top probability of exactly 1.0, those were 97.7% correct, and no threshold reached 98%. TF-IDF reached 98% on 79% of messages.
3. **Answers were stable, but location matters for latency.** On a 140-message subset, a repeat run, two shuffled option orders, and two rewordings changed 1 decision of 700, and it was a low-confidence one. From the Sydney function, the Jev call took 414 ms at p50 and 480 ms at p95. The same three questions sent directly from US West (Oregon) took 150 ms and 230 ms. These are 100 sequential requests from one location each, not a benchmark. TypeSafe states that most queries complete in about 100 ms; we didn't reach that from either location.

The self-serve answer separated help-article messages from staff-needed ones with an area under the ROC curve (AUROC) of 0.88. We didn't evaluate urgency, because neither dataset has urgency labels.

## Choosing a threshold

Jev returns two numbers per Choice answer: `probabilities` and `confidence`. Confidence rescales the top probability by the number of options, (p_max − 1/7) / (6/7) for seven queues, so 0 means a uniform spread and 1 means certainty. The app thresholds on `confidence`. If you add or remove a queue, the same top probability gives a different confidence, so re-tune the threshold.

*Table 2: Jev queue confidence thresholds on the 870 messages.*

| `confidence` threshold | Messages automated | Accuracy on automated |
|---|---|---|
| 0.64 | 95.6% | 95.3% |
| 0.8 (sample default) | 91.7% | 96.6% |
| 0.9 | 87.4% | 97.0% |
| 1.0 | 69.0% | 97.7% |

Pick the threshold from the cost of each mistake, not from a round number. Start strict, review what lands in the human queue, and lower it on evidence. Your queues and wording will move these numbers, so re-run the evaluation on a few hundred of your own labelled tickets. Give a costly queue such as `account_security` its own threshold in `app/src/routing.js`. Write borderline rules into the queue descriptions in `app/src/queues.json`, because that's the only place Jev learns them.

## Cost

These figures are for `ap-southeast-2` on October 14, 2026. Lambda rates came from the AWS Price List API: $0.0000279955 per vCPU-second, $0.0000037065 per GB-second, and $0.30 per million requests. Other Regions differ slightly; see [AWS Lambda pricing](https://aws.amazon.com/lambda/pricing/).

- **Lambda Web Functions:** memory is billed for an environment's whole life, including idle time, so cost depends on how busy each environment is. Our batch evaluation kept one environment busy and cost about **$0.0005 per 1,000 decisions**. The Step 6 burst left five environments running for about six minutes, an upper bound of about **$0.008 per 1,000 decisions**. A lower `--max-environments` keeps fewer idle environments.
- **Jev through OpenRouter (third party):** 898 input tokens per decision on average at $0.042 per million input tokens, with output tokens free. OpenRouter reported **$0.038 per 1,000 decisions**. All our runs together made about 2,300 decisions for about $0.09.
- **Other AWS charges:** the Secrets Manager secret ($0.40 per month, prorated) and CloudWatch Logs and custom metrics. For this walkthrough they come to a few cents.

## Prerequisites

- An AWS account with permissions to create Lambda Web Functions, IAM roles and inline policies, Secrets Manager secrets, and CloudWatch Logs queries.
- AWS CLI version 2.38 or later, which adds the `aws lambda-web` commands.
- Node.js 20 or later and npm. Web Functions run Node.js 24 on Arm64 (AWS Graviton). The app has no native dependencies, so you can install them on any machine.
- `curl` 7.75 or later, for `--aws-sigv4`. For the optional evaluation, Python 3.10 or later with `boto3` and `requests`.
- An [OpenRouter](https://openrouter.ai) API key. OpenRouter bills Jev usage separately from AWS.

At launch, Web Functions were available in US East (N. Virginia), US East (Ohio), US West (Oregon), Asia Pacific (Tokyo), and Europe (Ireland). We took the measurements in Asia Pacific (Sydney) on October 14, 2026, where the API was already available. We re-ran the deploy, authentication, and cleanup steps in US West (Oregon) on October 23, 2026. Check the [Lambda Web Functions documentation](https://docs.aws.amazon.com/lambda/latest/dg/web-functions.html) for your Region.

Every command reads the Region from `AWS_REGION`. Set it once:

```bash
export AWS_REGION=us-west-2   # or ap-southeast-2, or another Region with Web Functions
```

## Walkthrough

### Step 1: Review the decision call

Clone the repository and install the dependencies:

```bash
git clone https://github.com/opstodevops/jev-lambda-support-triage jev-triage && cd jev-triage
(cd app && npm ci --omit=dev)
```

The decision logic is in `app/src/jev.js`. The queue question passes the seven queue descriptions from `queues.json` as its options, and the TypeSafe SDK sends all three questions in one request. This excerpt is simplified from that file:

```javascript
const client = new TypeSafeClient({ apiKey, baseURL: 'https://openrouter.ai/api', defaultModel: 'jev-1.13' });

const questions = {
  queue: { type: 'choice', instructions: 'Which support queue should handle this customer message?', criteria: QUEUES },
  urgency: { type: 'score', instructions: 'How urgent is this message for the support team?', criteria: URGENCY_LEVELS },
  self_serve: { type: 'noul', instructions: 'Could a standard help-centre article fully resolve this, with no staff member needing to look at or act on the customer\'s account?' },
};

const { answers } = await client.systemOne({ state: message, questions });
// answers.queue      -> { choice: 'card_services', probabilities: {...}, confidence: 1 }
// answers.urgency    -> { score: 0.62, probabilities: {...}, confidence: 0.41 }
// answers.self_serve -> { noul: 0.39 }
```

The server is standard Express. The only Lambda-specific code splits `AWS_LAMBDA_HTTP_ENDPOINT` (for example `0.0.0.0:3000`) into the host and port that `app.listen()` binds.

### Step 2: Run it locally

Run the same server on your machine. For a local run, an environment variable is a reasonable place for the key:

```bash
cd app
read -rs -p "OpenRouter key: " OPENROUTER_API_KEY && export OPENROUTER_API_KEY
AWS_LAMBDA_HTTP_ENDPOINT=127.0.0.1:3000 node index.js

# In a second terminal:
curl -s localhost:3000/triage -H "content-type: application/json" \
  -d '{"message":"I think someone used my card without permission"}'
```

You should see `"queue": "account_security"`, an urgency near 2 (urgent), and a low self-serve probability, so the app routes the message to the security queue. Stop the server and return to the repository root.

### Step 3: Store the key and deploy the Web Function

The deployed function reads the key from Secrets Manager. Don't pass it as an environment variable, because anyone who can read the revision can read those in plaintext. The first command stores the key without putting it on a command line. The second packages `app`, uploads it to Amazon Simple Storage Service (Amazon S3), and creates the function, an execution role, and an HTTPS endpoint:

```bash
read -rs -p "OpenRouter key: " K && printf '%s' "$K" | aws secretsmanager create-secret \
  --name jev-triage/openrouter-api-key --secret-string file:///dev/stdin --query ARN --output text; unset K

aws lambda-web deploy --name jev-triage --code ./app --create \
  --auth-type IamAuth --max-environments 5 --rate-limit 100 --timeout-seconds 900 \
  --env JEV_SECRET_ID=jev-triage/openrouter-api-key --env NODE_ENV=production \
  --tags project=jev-triage
```

In our Sydney account, `deploy` finished in 17 seconds, and a later code update took 13 seconds. The output includes `endpointUrl` and `roleArn`. To ship a change, run the same command without `--create` and the endpoint options.

The scaling options cap your Jev spend. Each environment handles up to 64 concurrent requests by default. `--max-environments` caps environments and `--rate-limit` caps requests per second; requests above either get HTTP 429 and aren't queued. `--rate-limit` accepts set values such as 100, 200, and 1000, and rejected 50.

### Step 4: Let the function read the secret

`deploy` created a role named `awscli-lambdaweb-jev-triage`. Grant it read access to this one secret, replacing `111122223333` with your account ID:

```bash
aws iam put-role-policy --role-name awscli-lambdaweb-jev-triage --policy-name read-jev-secret \
  --policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"secretsmanager:GetSecretValue\",
  \"Resource\":\"arn:aws:secretsmanager:$AWS_REGION:111122223333:secret:jev-triage/openrouter-api-key-??????\"}]}"
```

The repository's `scripts/deploy.sh` runs Steps 3 and 4 in one go. It's safe to re-run, and it tags the secret and role.

### Step 5: Send a signed request

Because the endpoint uses `IamAuth`, an unsigned request returns HTTP 403. `curl` can sign the request with your AWS credentials. The signing service name is `lambda`, not `lambda-web`. Web Functions also require the `x-amz-content-sha256` header, which carries a hash of the body; without it, a signed request also gets 403.

```bash
DOMAIN=<endpointUrl without https://>
BODY='{"message":"My new card still has not arrived after two weeks"}'
eval "$(aws configure export-credentials --format env)"
curl -s "https://$DOMAIN/triage" --aws-sigv4 "aws:amz:$AWS_REGION:lambda" \
  --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
  ${AWS_SESSION_TOKEN:+-H "x-amz-security-token: $AWS_SESSION_TOKEN"} \
  -H "content-type: application/json" \
  -H "x-amz-content-sha256: $(printf '%s' "$BODY" | sha256sum | cut -d' ' -f1)" \
  -d "$BODY"
```

You should see a response like the following (shortened):

```json
{
  "decision": { "queue": "card_services", "confidence": 1, "urgency": 0.62, "selfServeP": 0.39, "jevMs": 690.6 },
  "action": { "action": "route_to_queue", "queue": "card_services", "reason": "confident queue" }
}
```

The caller needs the `lambda:InvokeWebFunctionEndpoint` IAM permission. Endpoints are reachable over the internet in both auth modes; `IamAuth` controls who can invoke them.

### Step 6: Watch one environment serve many requests

Send 200 requests with up to 50 open at once. This reuses the variables from Step 5 and counts the HTTP status codes:

```bash
curl -s --parallel --parallel-max 50 --aws-sigv4 "aws:amz:$AWS_REGION:lambda" \
  --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
  ${AWS_SESSION_TOKEN:+-H "x-amz-security-token: $AWS_SESSION_TOKEN"} \
  -H "content-type: application/json" \
  -H "x-amz-content-sha256: $(printf '%s' "$BODY" | sha256sum | cut -d' ' -f1)" \
  -d "$BODY" -o /dev/null -w "%{http_code}\n" "https://$DOMAIN/triage?n=[1-200]" | sort | uniq -c
```

The runtime writes a summary record per environment to CloudWatch Logs every minute. This CloudWatch Logs Insights query adds them up:

```bash
QID=$(aws logs start-query --log-group-name /aws/lambda/web/jev-triage \
  --start-time $(($(date +%s)-900)) --end-time $(date +%s) \
  --query-string 'filter type="platform.invokeSummary" | stats count_distinct(record.instanceId) as environments, sum(record.metrics.requests) as totalRequests, max(record.metrics.concurrentRequests) as peakConcurrency, sum(record.metrics.cpuTimeMs)/1000 as totalCpuSeconds' \
  --query queryId --output text) && sleep 10 && aws logs get-query-results --query-id "$QID"
```

In our Sydney run, all 200 requests returned HTTP 200 in 6.5 seconds. The query reported 5 environments and a peak of 15 concurrent requests on one environment. The fleet used 4.0 seconds of CPU over two minutes, including start-up and idle time. Wait a few minutes before you query, because our first attempts returned partial totals while new log streams were indexed.

Inside the app, the requests took 428 ms at p50 and 986 ms at p95. At our client in US West (Oregon), the slowest quarter took about 4 seconds. Those were the first requests on curl's 50 connections, whose TLS handshakes to Sydney averaged 2.2 seconds, so reuse connections in your client. A client close to the endpoint can exceed `--rate-limit 100`: in our us-west-2 re-run against the instant `/health` route, 128 of 200 requests got HTTP 429.

### Step 7 (optional): Evaluate on your own data

`POST /eval` streams a decision for each labelled message as Server-Sent Events (SSE). `eval/wf_client.py` signs the request and saves the results, and `eval/metrics.py` computes Table 1 and Figure 2. Our 870 messages took about 50 seconds with eight concurrent Jev calls. Replace `data/eval.jsonl` with your own labelled tickets; the README lists the commands.

## Considerations for production

This sample is for evaluation. Before you route real tickets with it, consider the following:

- **Access.** Grant `lambda:InvokeWebFunctionEndpoint` only to callers that need it, scoped to the endpoint ARN. Use the `lambda:Request/WebFunctionAuthType` condition key in a service control policy to block public endpoints. For a custom domain or AWS WAF, put Amazon CloudFront in front.
- **Infrastructure as code.** For a pipeline, use the CloudFormation types `AWS::Lambda::WebFunction`, `WebFunctionRevision`, and `WebFunctionEndpoint` instead of `deploy`, which creates a role and a shared bucket on the side.
- **Logs and alarms.** Lambda creates the log groups with no retention, so set one. Alarm on `HTTPResponseStatusClass5XX`, the throttle metrics, and the app's `JevLatency` metric in the `JevTriage` namespace.
- **Failure handling.** The SDK retries twice with a 10-second timeout per attempt. Set a deadline that fits your request path, and send messages to the human queue when Jev is unavailable.
- **Model versions.** Pin the model, and re-run the evaluation before you change it, because thresholds may not carry over.
- **Data handling.** Each message leaves AWS for OpenRouter and TypeSafe. Check their data handling against your privacy and data-residency requirements, and rotate the API key.

## Limitations

- **The data isn't your data.** Banking77 is public, and the queue mapping is ours. Each result comes from a single run, so treat differences of 1 to 2 points as noise.
- **Both services are weeks old.** We used SDK version 0.6.0, model `jev-1.13`, and AWS CLI 2.38. Expect changes.
- **We didn't test** non-English messages, prompt-injection attempts in message text, or multi-Region endpoints.

## Clean up

To avoid ongoing charges, run the cleanup script from the repository root with the same `AWS_REGION`:

```bash
./scripts/cleanup.sh
```

The script deletes the following, and took 16 seconds in our us-west-2 re-run:

- The Web Function and its endpoint.
- The `awscli-lambdaweb-jev-triage` role and its inline policies, including `awscli-lambdaweb-logs`, which `deploy` adds.
- The code packages under `lambda-web/jev-triage/` in the CLI bucket.
- Both log groups.
- The secret, without a recovery window. Add `--keep-secret` to keep it.

It leaves the bucket `awscli-lambdaweb-<account-id>-<region>-an`, which all CLI-deployed Web Functions in the Region share. Delete it if nothing else uses it. Finally, revoke the OpenRouter key if you created it for this walkthrough.

## Conclusion

A triage service is only as useful as its sense of when it might be wrong. On our test set, Jev automated 95.6% of messages at 95% accuracy or better, with no training data, for less than $0.04 per 1,000 decisions. Lambda Web Functions ran the Express app unchanged, and five environments served a 200-request burst with 4 seconds of CPU. Two limits stood out. No Jev threshold reached 98% accuracy, where a model trained on 10,099 examples did better. And calling from Sydney added about 264 ms per call compared with Oregon.

To try it, clone the [sample repository](https://github.com/opstodevops/jev-lambda-support-triage), replace `data/eval.jsonl` with a few hundred of your own labelled tickets, and pick the threshold that meets your accuracy target. To learn more, see the [Lambda Web Functions documentation](https://docs.aws.amazon.com/lambda/latest/dg/web-functions.html) and the [TypeSafe documentation](https://docs.typesafe.ai).

---

*References to third-party services or organizations in this post do not imply an endorsement, sponsorship, or affiliation between Amazon or AWS and the third party. TypeSafe, Jev, and OpenRouter are third-party products.*

## About the author

