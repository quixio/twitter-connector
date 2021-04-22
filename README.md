# twitter-connector
# Quix Twitter Connector

A Quix model can be used to bridge data from almost any source. In this case we'll be using a Twitter developer account to stream Tweet data with a view to running real time ML models on the feed later on.

To connect Quix to Twitter you'll need:

+ A Quix account and workspace (sign up at [https://portal.platform.quix.ai/self-sign-up](https://portal.platform.quix.ai/self-sign-up))
+ A Twitter developer account (sign up at [https://developer.twitter.com/en/apply-for-access](https://developer.twitter.com/en/apply-for-access))

Ensure you have the following info from your Twitter developer account:
+ Bearer Token

Head over to Quix and from within your Quix workspace do the following:

## Create a Topic to receive the data:
![image](https://user-images.githubusercontent.com/1799158/115437639-53892d80-a204-11eb-8250-1082c97c861c.png)

### Topic Id
Record the topic id for later, or remember to come back here when you need it

![TwitterConnector Get Topic Id](https://user-images.githubusercontent.com/1799158/115555627-9ea84c80-a2a7-11eb-86eb-d3614040d0cf.png)


## Create a new project and choose a language, I'm going with Python!
![image](https://user-images.githubusercontent.com/1799158/115437177-c514ac00-a203-11eb-88b0-1d8803dcb5de.png)

You will get some code with the basics filled in:
![image](https://user-images.githubusercontent.com/1799158/115437535-35233200-a204-11eb-9ea4-68d441fd50ea.png)

We'll be making HTTP requests from our Python project so be sure to add the requests requirement to the requirements.txt file
```
requests==2.25.1
```

It should also already have the 'quixstreaming' requirement

## Twitter Sample

The following code is based on a sample from the Twitter github for connecting to filtered streams [https://github.com/twitterdev/Twitter-API-v2-sample-code/blob/master/Filtered-Stream/filtered_stream.py](https://github.com/twitterdev/Twitter-API-v2-sample-code/blob/master/Filtered-Stream/filtered_stream.py)

### NOTE: 
The following code has some credentials and workspace specific details left blank. 
Please replace these with your own values.

These are the absolute minimum you will have to change. But you can also alter the Twitter rules (look for sample_rules) and the output stream name (look for stream.properties.name)

|What|Where to find it|
|----|----------------|
|Username|Found in your starter code|
|Password|Found in your starter code|
|OutputTopicId|See above|

```py
import requests
import os
import json
import pandas as pd
from datetime import datetime
from quixstreaming import *


certificatePath = "../certificates/ca.cert"

# use the values given in your starter project here
username = "[use your own value]"
password = "[use your own value]"
broker = "kafka-k1.quix.ai:9093,kafka-k2.quix.ai:9093,kafka-k3.quix.ai:9093"

security = SecurityOptions(certificatePath, username, password)
client = StreamingClient(broker, security)

#connect to the output topic
output_topic = client.open_output_topic("[use your own value]")

# define code to create the output stream
# you can change this to whatever you want
def create_stream():
    stream = output_topic.create_stream()
    stream.properties.name = "twitter_stream_results"
    return stream

# define the code to create the headers for the http connection
# dont forget to create the BEARER_TOKEN environment variable at deployment time
def create_headers(bearer_token):
    if bearer_token is None:
        raise Exception("Bearer token not set")

    headers = {"Authorization": "Bearer {}".format(bearer_token)}
    return headers

# define the code to get the existing rules from the twitter api
def get_rules(headers):
    response = requests.get(
        "https://api.twitter.com/2/tweets/search/stream/rules", headers=headers
    )
    if response.status_code != 200:
        raise Exception(
            "Cannot get rules (HTTP {}): {}".format(response.status_code, response.text)
        )
    print(json.dumps(response.json()))
    return response.json()

# code to delete the rules..
def delete_all_rules(headers, rules):
    if rules is None or "data" not in rules:
        return None

    ids = list(map(lambda rule: rule["id"], rules["data"]))
    payload = {"delete": {"ids": ids}}
    response = requests.post(
        "https://api.twitter.com/2/tweets/search/stream/rules",
        headers=headers,
        json=payload
    )
    if response.status_code != 200:
        raise Exception(
            "Cannot delete rules (HTTP {}): {}".format(
                response.status_code, response.text
            )
        )
    print(json.dumps(response.json()))

# code to create the rules..
# in this example were searching for tweets about F1...
def set_rules(headers, delete):
    # You can adjust the rules if needed
    sample_rules = [
        {"value": "#f1 OR f1", "tag": "F1 tweets"}
    ]
    payload = {"add": sample_rules}
    response = requests.post(
        "https://api.twitter.com/2/tweets/search/stream/rules",
        headers=headers,
        json=payload,
    )
    if response.status_code != 201:
        raise Exception(
            "Cannot add rules (HTTP {}): {}".format(response.status_code, response.text)
        )
    print(json.dumps(response.json()))

# here were going to get the stream and handle its output
# we'll do this by streaming the results into Quix
def get_stream(headers, set, quix_stream):
    response = requests.get(
        "https://api.twitter.com/2/tweets/search/stream", headers=headers, stream=True,
    )
    print(response.status_code)
    if response.status_code != 200:
        raise Exception(
            "Cannot get stream (HTTP {}): {}".format(
                response.status_code, response.text
            )
        )
    for response_line in response.iter_lines():
        if response_line:
            json_response = json.loads(response_line)
            print(json.dumps(json_response, indent=4, sort_keys=True))

            # get the data
            data = json_response["data"]
            # i want to store the tag in quix too so get the rules used to obtain this data
            matching_rules = json_response["matching_rules"]
            
            
# start everything going..
def main():
    bearer_token = os.environ.get("BEARER_TOKEN")
    headers = create_headers(bearer_token)
    rules = get_rules(headers)
    delete = delete_all_rules(headers, rules)
    set = set_rules(headers, delete)
    quix_stream = create_stream()
    get_stream(headers, set, quix_stream)


if __name__ == "__main__":
    main()
```

## Time to Deploy
To prepare for deployment within the Quix serverless environment we will:
+ Tag the code
+ Deploy the code
   + Configure the Twitter bearer token

## Create a Tag
Ensure you have saved the code then find the context menu on the right hand side to tag this version of the code

![image](https://user-images.githubusercontent.com/1799158/115556646-cba92f00-a2a8-11eb-908e-de896367a1a5.png)

![image](https://user-images.githubusercontent.com/1799158/115556838-01e6ae80-a2a9-11eb-8e63-9a72f0b73b05.png)

Give it any tag you want. Maybe "WorksFirstTime" ;-)

## Deploy
Now click Deploy

![image](https://user-images.githubusercontent.com/1799158/115556963-2478c780-a2a9-11eb-9c67-9d3994f3e7c7.png)

Select your 'version tag'. In my case it's "WorksFirstTime"

![image](https://user-images.githubusercontent.com/1799158/115557114-4b36fe00-a2a9-11eb-9082-e2e5f1dbbf07.png)

If you want to edit the name you can, but it's not required at the moment.

Now click the 'Variables' tab and enter BEARER_TOKEN as the name and use your twitter bearer token as the value

![image](https://user-images.githubusercontent.com/1799158/115557428-9b15c500-a2a9-11eb-97dc-122cc36a92a0.png)

Finally it's time to click 
![image](https://user-images.githubusercontent.com/1799158/115557497-aa950e00-a2a9-11eb-866e-da1f15e62b61.png)

## Run
The deployment should queue, build, start and run.
![image](https://user-images.githubusercontent.com/1799158/115557911-137c8600-a2aa-11eb-8fe5-90186f7ca86d.png)

You can check out the logs:
![image](https://user-images.githubusercontent.com/1799158/115558041-39098f80-a2aa-11eb-9041-bfe1c7c31f59.png)

![image](https://user-images.githubusercontent.com/1799158/115558187-648c7a00-a2aa-11eb-918b-c3ca1cd11348.png)

These show that data is being received from Twitter. (Or they might show details of an issue)

## See the results in Quix
Head over to the Data Catalogue, select the stream and click the visialize icon
![image](https://user-images.githubusercontent.com/1799158/115558822-057b3500-a2ab-11eb-9444-8735be02514c.png)

Select the parameters you are interested in

![image](https://user-images.githubusercontent.com/1799158/115558906-19bf3200-a2ab-11eb-9f73-704040176ba1.png)

![image](https://user-images.githubusercontent.com/1799158/115558979-2e032f00-a2ab-11eb-9e30-086b36890201.png)

This will show us the data in a graph, but seeing as this is mostly text data lets click on the Table tab

![image](https://user-images.githubusercontent.com/1799158/115559151-5723bf80-a2ab-11eb-8ecf-26f038fa4887.png)

Click "Load Data" and presto! You'll see all of the lovely Twitter data in Quix

![image](https://user-images.githubusercontent.com/1799158/115559272-791d4200-a2ab-11eb-8241-cb515bdf0bcb.png)





