---
title: AWS Lambda & Triggers Guide
---

# AWS Lambda: Bronze-to-Silver Data Pipeline Guide

## Overview of AWS Lambda and Triggers

AWS Lambda is a **serverless, event-driven compute service** – you deploy code as functions that run on demand without managing servers. Lambda automatically scales up to handle incoming events and you only pay for the compute time used. Developers simply write a *handler* function (in C#, Python, JavaScript, etc.) and AWS manages the runtime. For example, a simple C# handler for an S3 event might look like:

```csharp
public async Task FunctionHandler(S3Event evt, ILambdaContext context) {
    foreach (var rec in evt.Records) {
        string bucket = rec.S3.Bucket.Name;
        string key    = rec.S3.Object.Key;
        context.Logger.LogLine($"Processing {key} in bucket {bucket}");
        // transform data...
    }
}
```

Lambda functions are invoked by **events** or **triggers** from other AWS services. A common trigger is an **S3 event**: for example, when a new object is created in a bucket (the `s3:ObjectCreated:*` event), S3 can invoke your Lambda with the event data. The event payload is JSON; for instance:

```json
{
  "Records": [
    {
      "eventSource": "aws:s3",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": { "name": "my-bronze-bucket" },
        "object": { "key": "raw/data/file.csv" }
      }
    }
  ]
}
```

This tells the function which bucket and object triggered it.

Besides S3, Lambda can be triggered by many services. For example: **Amazon SQS** or **SNS** (messaging), **EventBridge** (scheduled or rule-based events), **API Gateway** (HTTP requests), **Kinesis/DynamoDB Streams**, **Step Functions** workflows, and more. Each trigger provides a JSON event specific to the source. For example, EventBridge can invoke Lambda on a cron schedule, and Step Functions can call Lambda as one step in a workflow. In the AWS Lambda console you can attach triggers under the “Designer” panel or by choosing **Add trigger**, then selecting the service and event (e.g. “S3” and “ObjectCreated (All)” with optional prefix filters).

## Inspecting Existing Lambda Functions

To examine a Lambda function in the AWS Console, open the **AWS Lambda** service and go to **Functions**. Click the function name to open its overview page. The console shows a **Designer** diagram with any configured triggers (e.g. an S3 bucket icon) and the function code/settings panels. If you don’t see triggers in the overview, check the **Aliases** tab – triggers are attached to specific versions or aliases (not always visible under `$LATEST`).

* **Code:** In the Lambda console, the **Code** or **Code + Test** tab shows the function’s code or deployment package. (For inline code you can edit here; for packaged deployments it shows a zip or a link). You can also see any Lambda layers.

* **Configuration:** Under **Configuration**, review the function’s settings. **General configuration** lists the runtime (e.g. .NET, Python, Node), handler name, memory size, and timeout. **Environment variables** (Configuration → Environment) store key settings like bucket names or API endpoints. Click *Edit* in the console to view or change them.

* **Permissions:** In Configuration → Permissions, you’ll see the **Execution role** (an IAM role ARN) that the function uses. Clicking this opens the IAM console where you can inspect policies. Ensure this role has only the permissions needed (e.g. `s3:GetObject`, `s3:PutObject`, `logs:PutLogEvents` etc.). Also check the *Resource-based policy* (Permissions tab) to see which services have permission to invoke this function (e.g. S3 invocation permission).

* **Triggers / Event Sources:** The **Designer** (or left panel) lists all triggers (event source mappings) for the function. For S3 triggers, it shows which bucket and event type (e.g. “s3\:ObjectCreated\:Put”). For stream-based sources (Kinesis/DynamoDB) it appears under **Event sources**. You can click each trigger to see or edit details.

* **Versions & Aliases:** If your function is versioned, check the **Versions** and **Aliases** tabs. Published versions are immutable snapshots of your function code/config. An alias (like “prod” or “dev”) is a pointer to a version. Triggers may be attached to a specific alias instead of `$LATEST`. Make note of which version/alias is active.

## Understanding Bronze-to-Silver Data Flow


In a medallion (“bronze-silver-gold”) architecture, data is organized into layers of quality. **Bronze** is the raw landing zone (e.g. S3 bucket) where data arrives unchanged from source. **Silver** is the cleansed, validated, and conformed layer built from Bronze. (A final Gold layer would contain aggregated or business-ready data.) For example, the *Bronze* layer might store raw CSV or JSON files; the *Silver* layer might store Parquet tables with enforced schema. The Silver layer data has “been matched, merged, conformed and cleansed” into an “enterprise view” of the entities.

Typical Lambda transformations from Bronze to Silver include:

* **Parsing and type conversion:** Read raw text (CSV/JSON) and convert fields into the proper types or schema.
* **Data cleaning:** Filter out invalid records, drop null fields, dedupe, and handle schema changes.
* **Enrichment:** Add or lookup additional information (e.g. joining in customer reference data or inserting timestamps).
* **Restructuring:** Rename fields, normalize formats, or change file formats (e.g. converting JSON/CSV to Parquet for analytics).
* **Validation:** Ensure required fields exist and data quality rules are met.

For example, a Python Lambda handler might load an S3 object, use `pandas` or Spark to clean it, then write the output to another S3 bucket or prefix designated as Silver. A C# Lambda could similarly use AWS SDK (`GetObjectAsync` on the Bronze bucket, apply transformations, then `PutObjectAsync` to the Silver bucket). AWS documentation provides examples of reading bucket/key from the S3 event and using an S3 client to get the object. In code you’ll often see:

```csharp
var bucket = evt.Records[0].S3.Bucket.Name;
var key    = HttpUtility.UrlDecode(evt.Records[0].S3.Object.Key);
var response = await _s3Client.GetObjectAsync(bucket, key);
// process the file...
await _s3Client.PutObjectAsync(silverBucket, newKey, processedStream);
```

This moves data from the *Bronze* bucket (`bucket`) to a *Silver* bucket, possibly under a new key. Remember to avoid recursion: if your function writes back to the same Bronze bucket, it can re-trigger itself. AWS warns to use separate buckets or prefixes to prevent infinite loops.

## Testing and Debugging Lambda Functions

* **Test Events in Console:** In the Lambda console, use the **Test** feature to simulate events. Click the **Test** tab (or **Code** tab’s test section) and either select a template or paste a JSON event. For an S3-triggered function, AWS provides sample S3 event templates. Give the test a name and JSON, then click **Test** to invoke the function with that payload. Check the **Execution result** panel for the response and any output.

* **Simulating S3 Events:** To test S3 triggers, you can use the AWS CLI or console to manually upload a test file to the Bronze bucket and watch Lambda run. Or in the Test event editor, use the “Amazon S3 Put” template (or paste a custom event). Ensure the JSON includes the correct bucket/key. For example, use the S3 event example from AWS docs (shown above) and adjust the bucket and key to a test object.

* **CloudWatch Logging:** Lambda automatically captures all console output and logging statements and sends them to **CloudWatch Logs**. In your code, write logs (e.g. `context.Logger.LogLine(...)` in C#, `print()` or `logging` in Python, `console.log()` in Node.js). These logs appear under the log group `/aws/lambda/<FunctionName>`. You can view logs from the Lambda console under the **Monitor** tab by clicking **View CloudWatch logs**. Each invocation creates log entries with a unique `RequestId`, along with automatic START/END/REPORT lines showing duration and memory. Use CloudWatch Logs Insights or filters to search and analyze logs. (Tip: Insert the request ID in your errors to trace a specific event.) Live troubleshooting is available via *CloudWatch Logs Live Tail*, which streams logs in real time.

* **Local Testing with SAM / IDE:** For rapid iteration, use the AWS SAM CLI or local tools. The `sam local invoke` command lets you run the Lambda function on your machine with a JSON event. For example:

  ```
  sam local invoke MyFunction -e event.json
  ```

  This uses a Docker container matching the Lambda runtime. You can even attach debuggers by running in debug mode (`sam local invoke -d 5858 ...`) and using an IDE. The **AWS Toolkit** plugins for Visual Studio, VS Code, or JetBrains IDEs integrate with SAM to enable breakpoints and step-through debugging. In Visual Studio for C#, the AWS Toolkit configures a local Lambda Test Tool, so you can set breakpoints in your handler and run the function as if it were local code. In any case, using `sam local start-api` (for HTTP-triggered Lambdas) or `sam local invoke` (for event triggers) is a powerful way to test logic before deploying.

* **End-to-End Testing:** After unit testing, do an end-to-end test by uploading sample data to the Bronze S3 bucket (or invoking the Step Function if used) and verifying that the Silver bucket gets the expected output. Monitor CloudWatch logs and the Lambda **Monitor** tab for errors or timeouts. Ensure the function’s IAM role has permissions (`s3:GetObject` on the Bronze bucket, `s3:PutObject` on the Silver bucket, plus CloudWatch Logs).

## Best Practices

* **Use Logging and Monitoring:** Instrument your code with logging (and consider AWS X-Ray for tracing) to diagnose issues. CloudWatch Metrics (invocation counts, errors, duration) and Logs are your first line of investigation. Tailor log verbosity (INFO vs DEBUG) via environment variables and ensure logs include context (e.g. S3 key, RequestId). Use CloudWatch Alarms to alert on high error rates or throttles.

* **Least Privilege IAM:** Give the Lambda’s execution role only the permissions it needs. For example, instead of `s3:*`, allow only `s3:GetObject` on the source bucket and `s3:PutObject` on the target bucket. Use the AWS-managed policy `AWSLambdaBasicExecutionRole` for logging, and custom policies for S3 access. Likewise, secure any downstream services (Databases, APIs) accessed by the function.

* **Use Environment Variables:** Store configuration (bucket names, prefixes, API endpoints) in environment variables rather than hard-coding. This makes it easy to switch between dev/test/prod setups. For example, have `BRONZE_BUCKET=my-bucket-bronze` and `SILVER_BUCKET=my-bucket-silver` as env vars that the code reads. Never store secrets directly; use AWS Secrets Manager or Parameter Store if needed.

* **Avoid Recursive Loops:** If the Lambda writes to an S3 bucket that also triggers it, you can get infinite invocation loops. To prevent this, use separate buckets or object key prefixes. For example, use a different “silver” bucket for output, or configure the S3 trigger with a prefix filter so only raw uploads (not processed files) invoke the function.

* **Idempotency:** Ensure your function can safely handle retries or duplicate events. Lambda may retry asynchronous invocations on failure. Design your code so that processing the same file twice does not corrupt data (e.g. by overwriting or checking for existing output). AWS recommends writing functions to be idempotent.

* **Test Safely:** Do development and testing in isolated environments. Use separate AWS accounts or at least separate S3 buckets for dev/test vs production. For production code, you can use Lambda versions and aliases (e.g. “dev” alias pointing to a test version, “prod” alias for stable) to control rollouts. AWS supports canary deployments and routing configurations for safe testing in production. When possible, test in a non-prod account with representative data. If testing in production, turn off triggers or point them to test data, and use small sample files.

* **Resource Configuration:** Set memory and timeout based on needs. More memory gives more CPU power (shortening run time). Use CloudWatch reports to see **Max Memory Used** and adjust. Do not set the timeout arbitrarily high – use just enough margin. Reserve concurrency or provisioned concurrency if you need consistent cold-start performance, but otherwise let Lambda scale automatically.  

  
---  
  

# AWS CLI Reference: Lambda Functions

## Invoke Lambda functions

Use the `aws lambda invoke` command to run a function. By default this is a **synchronous (RequestResponse)** invocation; specify `--invocation-type Event` for asynchronous.  Include your JSON payload (with AWS CLI v2 you must use `--cli-binary-format raw-in-base64-out` for inline JSON) and an output file. For example, to invoke **my-function** with a test event synchronously:

```bash
aws lambda invoke \
  --function-name my-function \
  --cli-binary-format raw-in-base64-out \
  --payload '{ "name": "Bob" }' \
  response.json
```

This saves the function output to `response.json`.  For asynchronous invocation, add `--invocation-type Event`:

```bash
aws lambda invoke \
  --function-name my-function \
  --invocation-type Event \
  --cli-binary-format raw-in-base64-out \
  --payload '{ "name": "Bob" }' \
  response.json
```

(An HTTP 202 status is returned).  **Gotchas:** In AWS CLI v2 you must use `--cli-binary-format raw-in-base64-out` when passing inline JSON (or use `file://payload.json`).  Quoting rules differ by shell (see AWS CLI docs).

## Deploy or update function code

You can create or update functions from local zip archives or from S3. Required options include function name, runtime (for zip), handler, and IAM role ARN.

* **Create a new function (zip file):** Use `aws lambda create-function`.  Example:

  ```bash
  aws lambda create-function \
    --function-name my-function \
    --runtime nodejs18.x \
    --zip-file fileb://my-function.zip \
    --handler my-function.handler \
    --role arn:aws:iam::123456789012:role/MyLambdaRole
  ```

  This uploads the ZIP `my-function.zip` (with your code and dependencies) and creates the function.
  For an S3-based deployment package, use the `--code S3Bucket=...,S3Key=...` syntax (in the same region) instead of `--zip-file`.  For example: `--code S3Bucket=my-bucket,S3Key=path/my-function.zip`.

* **Update existing function code (zip file):** Use `aws lambda update-function-code`.  Example:

  ```bash
  aws lambda update-function-code \
    --function-name my-function \
    --zip-file fileb://my-function.zip
  ```

  This replaces the code of the `$LATEST` version of **my-function** with the specified ZIP file.  The output shows the updated revision ID and other metadata.

* **Update code from S3:** Use `aws lambda update-function-code --s3-bucket <bucket> --s3-key <key>` to point at an S3 object. For example:

  ```bash
  aws lambda update-function-code \
    --function-name my-function \
    --s3-bucket my-bucket \
    --s3-key my-function.zip
  ```

  (The object must be in the same region, and you can optionally specify `--s3-object-version`.)  *See AWS docs on `--s3-bucket/--s3-key` usage.*

**Common issues:** Ensure your IAM role ARN is correct and has Lambda execution rights. Use the `fileb://` prefix for binary files. Don’t mix `--zip-file` and `--code`. If updating from S3, omit `--zip-file`.

## List and inspect functions

* **List functions:** `aws lambda list-functions` returns all functions in the account/region. Example:

  ```bash
  aws lambda list-functions
  ```

  This outputs a JSON array of function summaries (names, ARNs, runtimes, etc.). Use `--query` or text/table output for compact views.

* **Get function details:** `aws lambda get-function --function-name my-function` shows configuration and code location. Example:

  ```bash
  aws lambda get-function --function-name my-function
  ```

  The output includes the function’s ARN, runtime, handler, last modified, and a pre-signed URL for the .zip location.

* **Get function configuration:** `aws lambda get-function-configuration --function-name my-function[:version]` shows settings (memory, timeout, env vars, VPC, etc.). For example, to see version 2:

  ```bash
  aws lambda get-function-configuration --function-name my-function:2
  ```

  This lists the configuration fields (including environment variables and their KMS encryption if any).

* **List tags:** If you tag your function, use `aws lambda list-tags --resource <function-ARN>`. Example:

  ```bash
  aws lambda list-tags --resource arn:aws:lambda:us-west-2:123456789012:function:my-function
  ```

  Returns the tags map (key-value) attached to the function.

* **Get resource-based policy:** To view invoke permissions (e.g. grants to other services), use `aws lambda get-policy`. Example:

  ```bash
  aws lambda get-policy --function-name my-function
  ```

  This outputs the JSON policy document (with statements and principals) attached to the function.

*Tip:* Use `--output text` or `--query` to filter results. If a command complains about pagination, use `--no-paginate` or supply `--max-items`/`--starting-token`.

## Versions and aliases

Lambda allows versioning of code and named aliases pointing to versions.

* **Publish a version:** After updating code or config, run `aws lambda publish-version`. Example:

  ```bash
  aws lambda publish-version --function-name my-function
  ```

  This creates an immutable version (incrementing number) of the function from `$LATEST` (with an optional description or code hash check). The output shows the new version number.

* **List versions:** To list all published versions of a function, use:

  ```bash
  aws lambda list-versions-by-function --function-name my-function
  ```

  This returns a list of versions (including `$LATEST` and numbered versions) with metadata for each.

* **Create an alias:** An alias is a pointer to a specific version (for example, “LIVE” or “BETA”). To create:

  ```bash
  aws lambda create-alias \
    --function-name my-function \
    --name LIVE \
    --function-version 1 \
    --description "alias for live version"
  ```

  This makes alias **LIVE** point to version 1 of the function. (You can also configure traffic shifting weights in `--routing-config`.)

* **List aliases:** To see existing aliases for a function:

  ```bash
  aws lambda list-aliases --function-name my-function
  ```

  Returns all alias names, their ARNs, and the function version they reference.

* **Update alias:** To change which version an alias points to, use:

  ```bash
  aws lambda update-alias \
    --function-name my-function \
    --name LIVE \
    --function-version 3
  ```

  This repoints the alias **LIVE** to version 3 (you can also update alias description or routing).

* **Delete alias:** Remove an alias with:

  ```bash
  aws lambda delete-alias --function-name my-function --name LIVE
  ```

  This deletes the **LIVE** alias (no output).

**Tip:** Aliases are often used with API Gateway or other triggers. Remember that invoking a function with a version or alias requires including the qualifier (e.g. `my-function:2` or `my-function:LIVE`).

## Permissions (resource policies)

Lambda functions support resource-based IAM policies to allow other services or accounts to invoke them. Manage these with `add-permission` and `remove-permission`:

* **Add permission:** Grants an AWS principal (service or account) the right to invoke the function. For example, to allow SNS to invoke **my-function**:

  ```bash
  aws lambda add-permission \
    --function-name my-function \
    --statement-id sns \
    --action lambda:InvokeFunction \
    --principal sns.amazonaws.com
  ```

  This adds a statement (ID “sns”) allowing the SNS service to call the function. You can also specify `--source-arn` (or `--source-account`) to restrict it to a specific SNS topic, S3 bucket, etc.

* **Remove permission:** Deletes a permission statement by its ID. Use the same `--statement-id` you used when adding. Example:

  ```bash
  aws lambda remove-permission \
    --function-name my-function \
    --statement-id sns
  ```

  This removes the “sns” permission statement from the function policy.

After adding permissions, you can verify by running `get-policy` to see the updated statements. A common mistake is forgetting to set a unique `--statement-id`, or mismatching `--source-arn` so that invocations still fail.

## Event source mappings (triggers)

Lambda can be triggered by streams/queues via **event source mappings**. Use the following CLI commands for streaming sources like SQS, DynamoDB Streams, Kinesis, etc.:

* **Create mapping:** `aws lambda create-event-source-mapping` links a stream or queue to your function. Required: `--function-name`, `--event-source-arn`, and usually `--starting-position`. For example, to attach an SQS queue:

  ```bash
  aws lambda create-event-source-mapping \
    --function-name my-function \
    --event-source-arn arn:aws:sqs:us-west-2:123456789012:mySQSqueue \
    --batch-size 5
  ```

  This begins polling the SQS queue and sending up to 5 messages per Lambda invocation. For DynamoDB Streams or Kinesis, you must also specify `--starting-position` (e.g. `TRIM_HORIZON` or `LATEST`).

* **List mappings:** To list your event source mappings (by function or all):

  ```bash
  aws lambda list-event-source-mappings --function-name my-function
  ```

  This returns details (UUID, state, ARN) of each mapping attached to the function.

* **Delete mapping:** To remove a mapping, use the UUID from the list:

  ```bash
  aws lambda delete-event-source-mapping --uuid a1b2c3d4-5678-90ab-cdef-11111EXAMPLE
  ```

  This disables and deletes the mapping (state becomes Deleting).

* **Enable/disable mapping:** You can also enable or disable an existing mapping with `aws lambda update-event-source-mapping --uuid <id> --enabled true|false`.

**S3 event triggers:**  S3 bucket notifications are configured on the S3 side (not via Lambda’s event mappings). To trigger Lambda from S3, use the S3 CLI: for example, create a JSON notification config with a `LambdaFunctionConfigurations` section and run `aws s3api put-bucket-notification-configuration --bucket my-bucket --notification-configuration file://config.json`. (This is documented in the S3 CLI \[**put-bucket-notification-configuration**] but beyond Lambda’s own CLI.)

**Note:** Event source mappings currently support SQS, DynamoDB/Kinesis streams, Amazon MQ, MSK/Kafka, etc. S3 and SNS triggers use service-side notifications/permissions. Always ensure your function’s IAM role has permission to read the stream or queue.

