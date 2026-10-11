# Brew & Bean Email Triage API

## Overview

The Brew & Bean Email Triage API is a FastAPI service that classifies
customer emails into `order`, `feedback`, `support`, or `other`. It
returns a JSON category and confidence score. If confidence is below
`0.60`, the classifier returns `Uncertain`.

The project includes token authentication, rate limiting, structured
JSON logging, Docker support, automated tests, and a GitHub Actions
workflow for checks and Render deployment.

**Before delivery:** Replace the demo-video and live-URL placeholders
after you record and test them. Do not claim the service is live until
you have verified it.

## Setup

### Requirements

-   Python 3.11+
-   pip
-   Docker (optional)
-   OpenAI API key for model-powered classification

### Install

``` bash
python -m pip install -r requirements.txt
```

### Environment variables

Set these in your shell or hosting provider:

  Variable           Purpose
  ------------------ --------------------------------------------------
  `OPENAI_API_KEY`   Secret OpenAI API key; never commit it to GitHub
  `OPENAI_MODEL`     Model name; defaults to `gpt-4o-mini`
  `AUTH_TOKEN`       Secret token required by `/triage`

`.env.example` contains placeholders only. You do not need a real `.env`
file when configuring these values in Render. The current code reads
environment variables directly; it does not automatically load a `.env`
file.

### Run locally

``` bash
uvicorn app.main:app --host 127.0.0.1 --port 8000
```

-   Health: `http://127.0.0.1:8000/health`
-   Interactive API docs: `http://127.0.0.1:8000/docs`

If `OPENAI_API_KEY` is not configured, the current classifier uses local
fallback rules.

## Architecture

``` mermaid
flowchart TD
    A[Client or reviewer] -->|POST /triage + X-Auth-Token| B[FastAPI app]
    B --> C{Token valid?}
    C -->|No| D[401 Unauthorized]
    C -->|Yes| E[Rate limiter]
    E --> F[Email classification module]
    F --> G{OpenAI API key configured?}
    G -->|Yes| H[OpenAI model]
    G -->|No| I[Local fallback classifier]
    H --> J[Validate category and confidence]
    I --> J
    J --> K{Confidence below 0.60?}
    K -->|Yes| L[Return Uncertain]
    K -->|No| M[Return category]
    L --> N[JSON response and structured log]
    M --> N
    O[GitHub Actions] --> P[Black, Flake8, pytest]
    P -->|Successful main branch run| Q[Render deploy hook]
    Q --> B
```

### Components

-   `app/main.py`: API endpoints, authentication, rate limiting, JSON
    logging.
-   `app/triage.py`: classification, confidence threshold, OpenAI/local
    fallback.
-   `tests/test_main.py`: API tests.
-   `Dockerfile`: runs the API on port 8000 by default and respects the
    hosting `PORT`.
-   `.github/workflows/ci.yml`: runs Black, Flake8, and pytest, then
    triggers Render through a deploy hook.
-   `.env.example`: documents environment variable names without
    secrets.

## API Usage

### Health

``` bash
curl https://YOUR-RENDER-SERVICE.onrender.com/health
```

Expected:

``` json
{"status":"ok"}
```

### Classify an email

``` bash
curl -X POST "https://YOUR-RENDER-SERVICE.onrender.com/triage" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: YOUR_AUTH_TOKEN" \
  -d '{"email_text":"I want to know the status of my coffee order."}'
```

Example response (confidence is illustrative):

``` json
{
  "category": "order",
  "confidence": 0.9
}
```

Expected error cases: - Missing or incorrect token: `401 Unauthorized` -
Rate limit exceeded: `429 Too Many Requests` - Missing `AUTH_TOKEN`
configuration: `500` configuration error - Invalid request body: FastAPI
validation error

Never include real keys or tokens in the README, screenshots, demo
video, or public issues.

## Testing

Run:

``` bash
python -m pytest -q
python -m black --check app tests
python -m flake8 app tests
```

### Test plan

  -----------------------------------------------------------------------
  Test                    Procedure               Expected result
  ----------------------- ----------------------- -----------------------
  Health                  `GET /health`           HTTP 200 and
                                                  `{"status":"ok"}`

  Missing authentication  `POST /triage` without  HTTP 401
                          token                   

  Invalid token           Send incorrect token    HTTP 401

  Valid token             Send valid token and a  HTTP 200 with
                          clear order email       `category` and
                                                  `confidence`

  Empty email             Submit an empty string  Category `Uncertain`,
                                                  confidence `0.0`

  Rate limit              More than 10            Excess requests receive
                          requests/minute from    HTTP 429
                          one client IP           

  Black                   Run Black check         No formatting changes
                                                  required

  Flake8                  Run Flake8              No lint errors

  CI                      Push to `main` or open  GitHub Actions runs
                          a pull request          checks

  Deployment              Successful `main`       Render deployment is
                          workflow with           triggered
                          deploy-hook secret      
                          configured              
  -----------------------------------------------------------------------

Only state that tests passed after checking the actual CI output.

## Cost Estimate

Estimates below use GPT-4o mini text pricing of **\$0.15 per 1 million
input tokens** and **\$0.60 per 1 million output tokens**. Check current
pricing before budgeting.

### Average tokens per request (assumption)

  Item                                          Estimated average
  ------------------------------------------- -------------------
  Input (system prompt plus customer email)            250 tokens
  Output (category and confidence JSON)                 20 tokens
  Total                                                270 tokens

Actual usage varies with prompt and email length.

### Monthly API estimate

    Requests/month   Input tokens   Output tokens   Estimated API cost
  ---------------- -------------- --------------- --------------------
               100         25,000           2,000            \$0.00495
             1,000        250,000          20,000             \$0.0495
            10,000      2,500,000         200,000              \$0.495
            50,000     12,500,000       1,000,000              \$2.475

Formula:
`(input tokens / 1,000,000 × $0.15) + (output tokens / 1,000,000 × $0.60)`.

### Free hosting versus API usage

  -----------------------------------------------------------------------
  Component                                        Estimated monthly cost
  ------------------------------ ----------------------------------------
  Render free web service               \$0 while within free-plan limits

  OpenAI API                          Usage-based; not included in Render
                                                                  hosting

  Total at 1,000 requests/month       About \$0.05 API cost + \$0 hosting
  under the assumptions above    
  -----------------------------------------------------------------------

OpenAI API access and billing are separate from a ChatGPT subscription.
Render free services may spin down after inactivity and can take about a
minute to wake up. Review current policies: - [GPT-4o mini
pricing](https://developers.openai.com/api/docs/models/gpt-4o-mini) -
[Render free service limits](https://render.com/docs/free)

## Limitations

-   The local fallback uses simple keyword rules and can misclassify
    ambiguous emails.
-   Confidence values are not guaranteed to be calibrated probabilities.
-   The API classifies one email at a time; it does not connect to an
    inbox or route messages to staff.
-   Rate limiting is basic: 10 requests per minute per client IP.
-   Avoid logging full email contents without a privacy review.
-   Render free hosting can spin down during inactivity.
-   OpenAI API usage can incur charges.
-   CI deployment requires a GitHub secret named
    `RENDER_DEPLOY_HOOK_URL`.
-   A successful push does not prove CI or the live deployment works;
    check the logs.

## Demo

**Demo video:** https://youtube.com/shorts/9OaERlyt0Y0?si=BaTvasmfPLRT5zqQ

Record a short screen capture showing: 1. The live service or `/docs`
page. 2. A successful `POST /triage` request with a valid token (do not
reveal the token). 3. The JSON response. 4. A request without a token
returning `401`. 5. `/health` returning `{"status":"ok"}`.

Record with OBS or another screen recorder, upload to YouTube as
Unlisted, and replace the placeholder with the actual tested link. The
video link is not complete until you upload and test it.

**Live API URL:** `REPLACE_WITH_YOUR_RENDER_URL`\
**GitHub repository:**
https://github.com/jawad453/brew-bean-email-triage-api

## Sign-off

See [`signoff_template.md`](signoff_template.md) for the client review
and acceptance template.
