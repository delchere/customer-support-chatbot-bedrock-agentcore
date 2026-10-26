# Customer Support Chatbot with Amazon Bedrock AgentCore

<p align="center">
  <img src="first_project.png" alt="Udacity – Project Completion: Customer Support Chatbot" width="320">
</p>

A customer support chatbot for an online shop, built on **Amazon Bedrock AgentCore** (managed harness + MCP Gateway) with the **Amazon Nova Pro** model. Completed as part of Udacity's *Agentic AI Engineer* program.

The chatbot routes every customer message into one of three categories:

| Category | Behavior |
|---|---|
| **Bug report** | Collects the description, steps to reproduce and environment, one question at a time, then calls the `create_bug_report` tool and returns the ticket ID. |
| **Platform question** | Answers **only** from the FAQ (`online_shop_faq.md`). If the answer is not there, redirects to human support (1-800-555-0199). |
| **Other request** | Partnerships, disputes, complaints…: politely redirects to human support. |

## Architecture

```
User ──► chat.py ──► AgentCore Harness (Nova Pro + system prompt + FAQ)
                            │
                            ▼ (MCP tool)
                     AgentCore Gateway
                            │
                            ▼
              create_bug_report Lambda ──► DynamoDB (tickets)
```

- **CloudFormation** (`cloudformation-tool.yaml`): DynamoDB table, Lambda function and IAM roles (Lambda, Gateway, harness execution).
- **AgentCore Gateway** (`setup_gateway.py`): exposes the Lambda as the MCP tool `bugreports___create_bug_report` (AWS_IAM auth).
- **AgentCore Harness** (`create_harness.py`): managed agent with the system prompt (`system_prompt.txt`), into which the FAQ is injected via `{{FAQ}}`; greedy decoding for reliable tool calling.
- **Evaluation** (`cloudformation-testing.yaml`, `generate-eval-dataset.py`): S3 bucket + IAM role for **Bedrock Evaluations** in *LLM-as-a-judge* mode (bring-your-own-inference).

## Repository layout

| File | Purpose |
|---|---|
| `cloudformation-tool.yaml` | Tool infrastructure (DynamoDB, Lambda, IAM) |
| `cloudformation-testing.yaml` | Evaluation infrastructure (S3, Bedrock Evaluations role) |
| `create_bug_report.py` | Lambda code (mirrors the inline code in the template) |
| `setup_gateway.py` | Creates the Gateway and its Lambda target |
| `create_harness.py` | Creates / updates the harness |
| `system_prompt.txt` | System prompt and routing rules |
| `online_shop_faq.md` | FAQ knowledge base |
| `chat.py` | Interactive terminal chat with the harness |
| `harness-tests.json` | Test cases (bug, FAQ, other request) |
| `generate-eval-dataset.py` | Generates the JSONL dataset for Bedrock Evaluations |
| `cleanup_agentcore.py` | Deletes the AgentCore resources |
| `Customer Support Chatbot with Amazon Bedrock AgentCore-1.pdf` | Report with test screenshots |

## Getting started

Prerequisites: Python 3, configured AWS credentials, region `us-east-1`, access to the Amazon Nova Pro model.

```bash
pip install -r requirements.txt

# 1. Tool infrastructure
aws cloudformation deploy --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack --capabilities CAPABILITY_NAMED_IAM --region us-east-1

# 2. Gateway + harness
python setup_gateway.py
python create_harness.py

# 3. Chat with the bot
python chat.py
```

## Evaluation

```bash
aws cloudformation deploy --template-file cloudformation-testing.yaml \
  --stack-name bug-report-testing-stack --capabilities CAPABILITY_NAMED_IAM --region us-east-1

python generate-eval-dataset.py --tests-json harness-tests.json
```

The generated JSONL is uploaded to S3 and scored by a **Bedrock Evaluations** job (LLM-as-a-judge). Results are in `support-chatbot-eval-run-1/` and in the PDF report.

## Cleanup

```bash
python cleanup_agentcore.py
aws cloudformation delete-stack --stack-name bug-report-tool-stack --region us-east-1
aws s3 rm s3://<EvalDatasetBucketName> --recursive
aws cloudformation delete-stack --stack-name bug-report-testing-stack --region us-east-1
```

> `agentcore_config.json` (generated, contains the AWS account ARNs) is excluded from the repository via `.gitignore`.

## Author

Delchere Dontsa — [@delchere](https://github.com/delchere)
