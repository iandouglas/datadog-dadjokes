# Serving Up Dad Jokes with Google Cloud Run with DataDog monitoring

---

Today we're going to build a quick Flask application to serve "dad jokes" from Google Cloud Run and monitor usage with DataDog.



## Prior Knowledge / Installation

- comfort using a local terminal
- [Python 3.8 or newer](https://www.python.org/downloads/) installed (Mac OS X will have this already) and knowledge of "[virtual environments](https://docs.python.org/3/library/venv.html)"
- [Docker installed](https://docs.docker.com/get-docker/) on your system
- [Google Cloud command-line tooling](https://cloud.google.com/sdk/docs/install) installed, and logged in
- [DataDog account](https://app.datadoghq.com/signup), and [localized tools](https://app.datadoghq.com/signup/agent) installed


## Python Virtual Environment

Let's begin by setting up a virtual environment for our local work. Open your command-line terminal

```bash
$ python3 -m venv venv
$ source venv/bin/activate
```

At this point you should see the `(venv)` marker on your shell prompt.

Let's install the Flask framework to get started from the provided `requirements.txt` file:

```bash
(venv) $ pip3 install -r requirements.txt
```

This will install Flask v2.0.2 and its necessary dependencies.



## Set up a Google Cloud project

```bash
$ gcloud projects create datadog-dadjokes
$ gcloud auth configure-docker
$ gcloud components install docker-credential-gcr
$ gcloud alpha iam policies lint-condition
```

Answer "yes" if prompted to install additional modules.

To deploy this project, you will need Billing enabled on your Google Cloud account. Google offers a $300 credit for first-time users on their platform.

Our usage will be extremely minimal and should not impact any free quota levels at Google Cloud to get started.

Next, we need to set up IAM and Billing:

- visit https://cloud.google.com/docs/authentication/getting-started
- click on "Go to Create service account"
- choose your project from the "recent projects" screen
- fill in the following details:
  - Service account details: "datadog dad jokes"
  - click "Create and Continue"
- For access grants, scroll to "Cloud Run" and choose "Cloud Run Admin" from the submenu
- click Done

On the Service Accounts for your project, you should see an "email address" that looks something like this:

`datadog-dad-jokes@voltaic-talent-338006.iam.gserviceaccount.com`

Your project name will differ.

Next, let's link the IAM to our project. First, we need to get our Google Cloud project ID:

```bash
$ gcloud projects list
PROJECT_ID             NAME              PROJECT_NUMBER
datadog-dadjokes       datadog-dadjokes  625094981718
```

And we will input that project ID number and the "email address" from above to set our IAM policy:

```bash
$ gcloud projects add-iam-policy-binding 625094981718 --member "serviceAccount:datadog-dad-jokes@voltaic-talent-338006.iam.gserviceaccount.com" --role "roles/owner"
Updated IAM policy for project [625094981718].
bindings:
- members:
  - serviceAccount:datadog-dad-jokes@voltaic-talent-338006.iam.gserviceaccount.com
  - user:wil@wildouglas.com
  role: roles/owner
etag: BwXVXRDhaTE=
version: 1
```

Visit your Google Console, go into your Billing, and be sure you click on the "Activate" button at the very top of the screen if you are activating the free $300 plan.

You MUST link your project to a billing account to deploy later.


## Python Code to get started

We're going to begin by writing a very simple Flask application that reads a file of data jokes in JSON format, and return a random joke from the file.

Let's store this in a filename called `dadjokes.py`:

```python
from flask import Flask
import flask
import os
import random
import json


jokes = []
with open('jokes.json', 'r') as joke_data:
  jokes = json.load(joke_data)


app = Flask(__name__)

@app.route('/')
def get_a_joke():
  jokes_index = random.randint(0,len(jokes)-1)
  random_joke = jokes[jokes_index]
  return random_joke


if __name__ == "__main__":
  print(f'loaded {len(jokes)} jokes')
  port = int(os.environ.get('PORT', 5000))
  app.run(debug=True, host='0.0.0.0', port=port)

```

Let's run it locally to ensure this works in our browser:

```bash
(venv) $ python3 dadjokes.py
loaded 205 jokes
 * Serving Flask app 'dadjokes' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://192.168.1.2:5000/ (Press CTRL+C to quit)
 * Restarting with stat
loaded 205 jokes
 * Debugger is active!
 * Debugger PIN: 140-240-166
```

Your IP address may differ, but the output is showing us the URL to hit in our browser. In this case, visiting `http://192.168.1.2:5000/` should show us a random "dad joke" and hitting refresh should load a different joke each time.


## Docker container

Let's build the container:

```bash
$ docker image build -t datadog_jokes .
```

Then we'll run the container locally:
```bash
$ docker run -p 5001:5000 -d datadog_jokes
```

This will run a LOCAL connection on port 5001 and connect to port 5000 on the Docker instance.

If we view `http://localhost:5001/` in the browser, we should see a random dad joke in the browser.


## Deploying to Google

Next, let's push our code to Google.

We're going to start with the [documentation](https://cloud.google.com/run/docs/deploying-source-code) about deploying from source code, and not from an uploaded Docker image.

```bash
$ gcloud run deploy datadog-dadjokes --source .
```

You may be prompted to enable APIs, answer "yes". You MUST have billing enabled to move forward from here.

You may be prompted for a region where to run your project. Choose whatever is convenient for you.

You may also be prompted (again) to enable the API permission, answer "yes".

Next, you may be prompted to set your docker container as deployable on Google, answer "yes".

Then you will be prompted if unauthenticated access can be used, answer "yes" again.

Your code should be deploying at this point. I was asked to enable API access again.

Eventually we see a success like

```text
Operation "operations/acf.p2-625094981718-23d3beb5-f2c5-4bfe-9a01-1b8b53852e49" finished successfully.
```

Toward the very end of the output, you should see something like this:

```text
Service [datadog-dadjokes] revision [datadog-dadjokes-00001-did] has been deployed and is serving 100 percent of traffic.
Service URL: https://datadog-dadjokes-7aeteatfua-uc.a.run.app
```

Let's hit that URL in our browser and see our dad jokes live on the internet!


## Wait, what about DataDog?

Finally, let's ping DataDog when someone loads a dad joke.

We'll start with our dashboard here:
- https://app.datadoghq.com/event/explorer

We'll begin by getting our API Key from this URL:
- https://app.datadoghq.com/organization-settings/api-keys

The DataDog Python library has already been installed as part of our `requirements.txt` file.

We have to visit our dashboard to install the Python integration:
https://app.datadoghq.com/account/settings#integrations


We'll add our block of code under our other imports for when our Python script is getting started:

```python
from datadog_api_client.v1 import ApiClient, Configuration
from datadog_api_client.v1.api.events_api import EventsApi
from datadog_api_client.v1.model.event_create_request import EventCreateRequest

configuration = Configuration(
    host = "https://api.datadoghq.com"
)

configuration.api_key['apiKeyAuth'] = 'YOUR_API_KEY'
configuration.api_key['appKeyAuth'] = 'YOUR_APP_KEY'
datadog = ApiClient(configuration)
```

And then within our `get_a_joke` method, we'll add our DataDog event tracking:

```python

@app.route('/')
def get_a_joke():
  jokes_index = random.randint(0,len(jokes)-1)
  random_joke = jokes[jokes_index]

  title = "Someone read a dad joke!"
  text = f"visitor IP: {flask.request.remote_addr}"
  body = EventCreateRequest(
    title=title,
    text=text,
    tags=["joke"],
  )
  api_instance = EventsApi(datadog)
  api_response = api_instance.create_event(body=body)
  print(api_response)

  return random_joke
```

We can run the code manually with `python3 dadjokes.py` to verify this works, or run it with our Docker container.

But let's re-deploy to Google:

```bash
$ gcloud run deploy datadog-dadjokes --source .
```

We should see our events showing up on our dashboard at https://app.datadoghq.com/event/explorer when we access our web application on Google Cloud, and our remote IP address should show up as part of the event.

