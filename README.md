# Power of Math — End-to-End AWS Web Application

A serverless web application built entirely on AWS. Users enter a base and an exponent, hit Calculate, and get the result back in the browser. Every calculation is stored in a DynamoDB table. The project walks through five AWS services and shows how they wire together into a working application.

This repo documents the build from the tutorial video: [Architect and Build an End-to-End AWS Web Application from Scratch, Step by Step](https://www.youtube.com/watch?v=7m_q1ldzw0U).

---

## Architecture

```
Browser (Amplify-hosted HTML)
        |
        | HTTP POST (JSON)
        v
  API Gateway (REST API)
        |
        | Invoke
        v
   AWS Lambda (Python)
        |
        | boto3 PutItem
        v
    DynamoDB (NoSQL table)
```

**Services used:**

| Service | Role |
|---|---|
| AWS Amplify | Hosts and serves the static frontend |
| AWS Lambda | Runs the math calculation logic |
| Amazon API Gateway | Exposes Lambda as an HTTPS endpoint |
| Amazon DynamoDB | Stores each calculation result |
| AWS IAM | Grants Lambda permission to write to DynamoDB |

---

## Prerequisites

- An AWS account (free tier is sufficient for this project)
- A text editor
- Basic familiarity with the AWS Console

No CLI setup or local environment is required — everything is done through the AWS Management Console.

---

## Step 1 — Create and Host the Initial Web Page with AWS Amplify

Before adding any backend logic, we deploy a placeholder HTML page so that Amplify hosting is configured and ready to receive updates later.

**Create the file:**

`frontend/index-v1.html`

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>To the Power of Math!</title>
</head>
<body>
    To the Power of Math!
</body>
</html>
```

**Deploy to Amplify:**

1. Open the [AWS Amplify Console](https://console.aws.amazon.com/amplify/home).
2. Under **Amplify Hosting**, click **Get started**.
3. Choose **Deploy without Git provider** (manual deployment).
4. Give your app a name (e.g., `PowerOfMath`) and set the environment name (e.g., `dev`).
5. Zip the HTML file — it must be uploaded as a `.zip`. Name it `index.zip`.
6. Drag and drop `index.zip` into the file upload area.
7. Click **Save and Deploy**.

Once the deployment finishes, Amplify gives you a public URL like `https://dev.d1abc123.amplifyapp.com`. Opening it should display "To the Power of Math!".

---

## Step 2 — Create the Lambda Function

Lambda will handle the actual math. We write a simple Python function that reads a `base` and `exponent` from the event, calculates the result, and returns it.

**Function name:** `PowerOfMathFunction`  
**Runtime:** Python 3.11 (or the latest Python 3.x available)

1. Open the [Lambda Console](https://console.aws.amazon.com/lambda/home).
2. Click **Create function**.
3. Select **Author from scratch**.
4. Set the function name to `PowerOfMathFunction`.
5. Set the runtime to **Python 3.11**.
6. Click **Create function**.

**Paste in this code** (`lambda/lambda_function_v1.py`):

```python
import json
import math

def lambda_handler(event, context):
    mathResult = math.pow(int(event['base']), int(event['exponent']))

    return {
        'statusCode': 200,
        'body': json.dumps('Your result is ' + str(mathResult))
    }
```

Click **Deploy** to save the function.

---

## Step 3 — Test the Lambda Function

Before connecting anything else, verify the function works on its own.

1. In the Lambda console, click **Test**.
2. Create a new test event. Use this payload (`lambda/test_event.json`):

```json
{
    "base": 2,
    "exponent": 10
}
```

3. Give the test event a name (e.g., `MyMathTest`) and click **Save**.
4. Click **Test** again.

You should see a successful execution with the response body: `"Your result is 1024.0"`.

---

## Step 4 — Create the REST API with API Gateway

API Gateway will sit between the browser and Lambda, giving us a public HTTPS endpoint to call.

1. Open the [API Gateway Console](https://console.aws.amazon.com/apigateway/home).
2. Click **Create API**, then select **REST API** and click **Build**.
3. Choose **New API**.
4. Set the API name (e.g., `PowerOfMathAPI`) and click **Create API**.

**Add a POST method:**

1. In the **Resources** panel, select the root resource (`/`).
2. From the **Actions** menu, choose **Create Method**.
3. Select **POST** from the dropdown and click the checkmark.
4. Set **Integration type** to **Lambda Function**.
5. Type `PowerOfMathFunction` in the Lambda Function field.
6. Click **Save**, then **OK** when prompted to grant API Gateway permission to invoke Lambda.

**Enable CORS:**

1. Make sure the POST method is selected.
2. From the **Actions** menu, choose **Enable CORS**.
3. Leave the default settings and click **Enable CORS and replace existing CORS headers**.
4. Click **Yes, replace existing values** when prompted.

**Deploy the API:**

1. From the **Actions** menu, choose **Deploy API**.
2. For **Deployment stage**, select **[New Stage]**.
3. Set the stage name to `dev`.
4. Click **Deploy**.

After deployment, copy the **Invoke URL** shown at the top of the stage editor. It looks like `https://abc123.execute-api.us-east-1.amazonaws.com/dev`. You will need this in Step 7.

---

## Step 5 — Create the DynamoDB Table

Each calculation result will be persisted in DynamoDB so there is a record of every query.

1. Open the [DynamoDB Console](https://console.aws.amazon.com/dynamodb/home).
2. Click **Create table**.
3. Set the **Table name** to `PowerOfMathDatabase`.
4. Set the **Partition key** to `ID` (String).
5. Leave everything else at defaults and click **Create table**.

Once the table is created, click on it and expand **Additional info** under the **Overview** tab. Copy the **ARN** — it looks like `arn:aws:dynamodb:us-east-1:123456789:table/PowerOfMathDatabase`. You will need it in the next step.

---

## Step 6 — Grant Lambda Permission to Write to DynamoDB

By default, Lambda cannot touch DynamoDB. We attach an inline IAM policy to the Lambda execution role.

1. Go back to your Lambda function (`PowerOfMathFunction`).
2. Click the **Configuration** tab, then **Permissions** in the left panel.
3. Under **Execution role**, click the role name link. This opens IAM in a new tab.
4. In IAM, click **Add permissions** → **Create inline policy**.
5. Switch to the **JSON** editor and paste the policy below.
6. Replace `YOUR-DYNAMODB-TABLE-ARN` with the ARN you copied in Step 5.

`iam/dynamodb-policy.json`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:DeleteItem",
                "dynamodb:GetItem",
                "dynamodb:Scan",
                "dynamodb:Query",
                "dynamodb:UpdateItem"
            ],
            "Resource": "YOUR-DYNAMODB-TABLE-ARN"
        }
    ]
}
```

7. Click **Next**, give the policy a name (e.g., `PowerOfMathDynamoDBPolicy`), and click **Create policy**.

---

## Step 7 — Update Lambda to Write Results to DynamoDB

Now that Lambda has the right permissions, update the function code to actually write to the table.

**Replace the function code** with the version in `lambda/lambda_function.py`:

```python
import json
import math
import boto3
from time import gmtime, strftime

dynamodb = boto3.resource('dynamodb')

# Make sure this matches the DynamoDB table name you created
table = dynamodb.Table('PowerOfMathDatabase')

now = strftime("%a, %d %b %Y %H:%M:%S +0000", gmtime())

def lambda_handler(event, context):
    mathResult = math.pow(int(event['base']), int(event['exponent']))

    response = table.put_item(
        Item={
            'ID': str(mathResult),
            'LatestGreetingTime': now
        }
    )

    return {
        'statusCode': 200,
        'body': json.dumps('Your result is ' + str(mathResult))
    }
```

Click **Deploy** to save the updated function.

Run the test again (same payload from Step 3). After it succeeds, go to the DynamoDB console, open the `PowerOfMathDatabase` table, and check **Explore table items** — you should see a row with the result.

---

## Step 8 — Update the Frontend to Call API Gateway

Now we update the HTML page to actually send the user's inputs to the API and display the result.

**Replace the placeholder HTML** with the final version in `frontend/index.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>To the Power of Math!</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 40px;
        }
        h1 { color: #333; }
        input {
            margin: 5px;
            padding: 8px;
            font-size: 16px;
            width: 200px;
        }
        button {
            margin-top: 10px;
            padding: 10px 20px;
            font-size: 16px;
            cursor: pointer;
        }
        #result {
            margin-top: 20px;
            font-size: 20px;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <h1>To the Power of Math!</h1>

    <label>Base:</label>
    <input id="base" type="number" placeholder="Enter base">

    <label>Exponent:</label>
    <input id="exponent" type="number" placeholder="Enter exponent">

    <button onclick="callAPI()">Calculate</button>

    <p id="result"></p>

    <script>
        // Replace this with your actual API Gateway Invoke URL
        const API_ENDPOINT = "YOUR-API-GATEWAY-INVOKE-URL";

        function callAPI() {
            const base = document.getElementById("base").value;
            const exponent = document.getElementById("exponent").value;

            fetch(API_ENDPOINT, {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ base: base, exponent: exponent })
            })
            .then(response => response.json())
            .then(data => {
                document.getElementById("result").innerHTML = data.body;
            })
            .catch(error => {
                console.error("Error:", error);
                document.getElementById("result").innerHTML =
                    "Error calling the API. Check the console for details.";
            });
        }
    </script>
</body>
</html>
```

**Before zipping:** replace `YOUR-API-GATEWAY-INVOKE-URL` with the Invoke URL you copied at the end of Step 4.

---

## Step 9 — Redeploy the Updated Frontend to Amplify

1. Zip the updated `index.html` into a new `index.zip`.
2. Open the Amplify Console and go to your app.
3. Click **Deploy** (or drag and drop the new zip file).
4. Wait for the deployment to go green.

Open the Amplify URL, enter a base and exponent, and click **Calculate**. The result should appear on the page. Check DynamoDB to confirm the item was written.

---

## Cleaning Up

To avoid ongoing charges, delete the resources in this order:

1. **Amplify** — Open the app, click **Actions** → **Delete app**.
2. **API Gateway** — Select the API, click **Actions** → **Delete**.
3. **Lambda** — Select the function, click **Actions** → **Delete**.
4. **DynamoDB** — Select the table, click **Delete**.
5. **IAM** — Go to the role that was auto-created for Lambda, find the inline policy you added, and delete it (or delete the role if it was created solely for this project).

---

## Repository Structure

```
.
├── frontend/
│   ├── index-v1.html          # Initial placeholder page (Step 1)
│   └── index.html             # Final page with API call (Step 8)
├── lambda/
│   ├── lambda_function_v1.py  # Lambda without DynamoDB (Steps 2-4)
│   ├── lambda_function.py     # Lambda with DynamoDB (Steps 7+)
│   └── test_event.json        # Test payload for the Lambda console
└── iam/
    └── dynamodb-policy.json   # Inline policy for Lambda execution role
```

---

## Common Issues

**CORS error in the browser console**

Make sure you enabled CORS in API Gateway (Step 4) and redeployed the API after enabling it. The deployment step is easy to miss.

**Lambda times out or returns a permission error**

Check that the IAM inline policy (Step 6) has the correct DynamoDB table ARN. Even a small typo in the ARN will cause an authorization failure.

**DynamoDB table not found**

The table name in `lambda_function.py` must exactly match what you created in the DynamoDB console, including capitalization. The code uses `PowerOfMathDatabase`.

**Amplify showing old content**

Amplify caches aggressively. After redeploying, do a hard refresh (Ctrl+Shift+R or Cmd+Shift+R) or wait a minute and try again.
