# Lab 7 - Put it all together

<img src="https://github.com/openfaas/media/raw/master/OpenFaaS_Magnet_3_1_png.png" width="500px"></img>

We're going to create a GitHub bot with OpenFaaS functions named `issue-bot`.

The job of issue-bot is to triage new issues by analysing the sentiment of the "description" field, it will then apply a label of *positive* or *review*. This will help the maintainers with their busy schedule so they can prioritize which issues to look at first.

![](./diagram/issue-bot.png)

## Get a GitHub account

* Sign up for a [GitHub account](https://github.com) if you do not already have one.

* Create a new repository and call it *bot-tester*

## Set up a tunnel with ngrok

You will need to receive incoming webhooks from GitHub. In production you will have a clear route for incoming traffic but within the constraints of a workshop we have to be creative.

Head over to ngrok.com and download the tool and unzip it.

Run this on your local computer:

```
$ ngrok http 8080
```

You will be given a URL that you can access over the Internet, it will connect directly to your OpenFaaS API Gateway.

Test the URL such as http://fuh83fhfj.ngrok.io

```
$ faas-cli list --gateway http://fuh83fhfj.ngrok.io/
```

## Create an webhook receiver `issue-bot`

```
$ faas new --lang python issue-bot --prefix="<your-docker-username-here>"
$ faas build -f ./issue-bot.yml
$ faas push -f ./issue-bot.yml
```

Now edit the function's YAML file and add an environmental variable of `write_debug: true`.

* Deploy the function

```
$ faas deploy -f ./issue-bot.yml
```

## Receive webhooks from GitHub

Log back into GitHub and navigate to your repo *bot-tester*

Click Settings -> Webhooks

Now enter the URL you were given from Ngrok adding `/function/issue-bot` to the end, for example:

```
http://fuh83fhfj.ngrok.io/function/issue-bot
```

For content-type pick: application/json

And select "Send me everything"

## Check it worked

Now go to GitHub and create a new issue. Type "test" for the title and description.

Check how many times the function has been called:

```
$ faas list
Function    Invocations
issue-bot   1
```

Each time you create an issue the count will increase. You can see the payload sent via GitHub by typing in `docker service logs -f issue-bot`.

## Analyse new Issues on GitHub


### Deploy Sentiment Analysis function

In order to use this issue-bot function, you will need to deploy the Sentiment Analysis function first.
This is a python function that provides a rating on sentiment positive/negative (polarity -1.0-1.0) and subjectivity provided to each of the sentences sent in via the TextBlob project.

You can checkout the function from [faas/sample-functions](https://github.com/openfaas/faas/tree/master/sample-functions).

Deploy with: 

```
curl -s http://fuh83fhfj.ngrok.io/system/functions --data-binary \
'{ 
   "service": "sentimentanalysis",
   "image": "functions/sentimentanalysis",
   "envProcess": "python ./handler.py",
   "network": "func_functions"
   }'
```

and test

```
# curl http://fuh83fhfj.ngrok.io/function/sentimentanalysis -d "I am really excited to participate in the OpenFaaS workshop."
Polarity: 0.375 Subjectivity: 0.75

# curl http://fuh83fhfj.ngrok.io/function/sentimentanalysis -d "The hotel was clean, but the area was terrible"; echo
Polarity: -0.316666666667 Subjectivity: 0.85
```

### Update your `issue-bot` function

Open `issue-bot/handler.py` and paste this code:

```
import requests, json, os, sys

def handle(req):

    event_header = os.getenv("Http_X_Github_Event")

    if not event_header == "issues":
        sys.exit(1)
        return

    gateway_hostname = os.getenv("gateway_hostname", "gateway")

    payload = json.loads(req)

    if not payload["action"] == "opened":
        return

    #sentimentanalysis
    res = requests.post('http://' + gateway_hostname + ':8080/function/sentimentanalysis', data=payload["issue"]["title"]+" "+payload["issue"]["body"])

    print(res.json())
```

Update `requirements.txt` with 

```
requests
```

**TODO** - add more details about what the code does

### Build and Deploy:

Use the CLI to build and deploy the function:

```
faas build -f issue.yml & faas push -f issue-bot.yml & faas deploy -f issue-bot.yml
```

Now create new issues in the `bot-tester` repo and check the Response from the repo Webhook.
**TODO** Add more details


## Apply labels via the GitHub API

### Create Github Auth token

Go to your GitHub profile -> Settings/Developer settings/Personal access tokens and generate new token.

Copy the contents of `env.example.yml` to `env.yml` and update `auth_token` value with the new generated token.

Update `repo` in `env.yml` with the `bot-tester` repository.


### Update code

Open `issue-bot/handler.py` and update the code with:

```
from github import Github

# ... leave the old code here

# positive_threshold
    positive_threshold = float(os.getenv("positive_threshold", "0.2"))

    g = Github(os.getenv("auth_token"))
    repo = g.get_repo(os.getenv("repo"))
    issue = repo.get_issue(payload["issue"]["number"])

    has_label_positive = False
    has_label_review = False
    for label in issue.labels:
        if label == "positive":
            has_label_positive = True
        if label == "review":
            has_label_review = True

    if res.json()['polarity']  >  positive_threshold and not has_label_positive:
        issue.set_labels("positive")
    elif not has_label_review:
        issue.set_labels("review")

    print(res.json())
```
**TODO** Add more details about the code

Update `requirements.txt` with 

```
PyGithub
```

### Build and Deploy:

Use the CLI to build and deploy the function:

```
faas build -f issue.yml & faas push -f issue-bot.yml & faas deploy -f issue-bot.yml
```

Now create new issues in the `bot-tester` repo and check the labels.
**TODO** more details


Now return to the [main page for Q&A](./README.md).
