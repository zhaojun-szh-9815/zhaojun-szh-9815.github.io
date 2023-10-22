---
title: Asynchronous OpenAI API Request
date: 2023-10-22 00:42:00 -0500
categories: [Work Daily, Data Science]
tags: [python, openai, gpt, api, asynchronous, parallel]     # TAG names should always be lowercase
---

# Asynchronous OpenAI API Request

## Setup

- Python environment
    - Python 3.7.1 or newer
    - Virtual environment (optional)
    - pip install --upgrade openai
- OpenAI API Key
    - create key on OpenAI account or by command
    - setup environment variable "OPENAI_API_Key"
    - or use ```openai.api_key = "your_key"```

## A Simple OpenAI API call

```py
import os
import openai
openai.api_key = os.getenv("OPENAI_API_KEY")

completion = openai.ChatCompletion.create(
  model="gpt-3.5-turbo",
  messages=[
    {"role": "system", "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."},
    {"role": "user", "content": "Compose a poem that explains the concept of recursion in programming."}
  ]
)
print(completion.choices[0].message)
```

## Enable retry to deal with API Limit

Image you have a list of data, for each data, you want to ask GPT the same question it.<br>
You can do this task as follow

- Import [tenacity](https://tenacity.readthedocs.io/en/latest/), which allows the program resend to request many times and wait some time before resend
```py
from tenacity import (
    retry,
    stop_after_attempt,
    wait_random_exponential,
)
```

- Define a function to make API request with annotation @retry, the parameter 'messages' is a list just like the variable 'message' in example above
```py
@retry(wait=wait_random_exponential(min=30, max=60), stop=stop_after_attempt(5))
def request_to_gpt(messages: list):
    return openai.ChatCompletion.create(
          model="gpt-3.5-turbo",
          messages=messages
    )
```

- Define a function to load all data and make API request
```py
def analyze_data(data: str):
    messages = []
    system_message = "YOUR QUESTION"
    messages.append({"role":"system","content":system_message})
    messages.append({"role":"user","content": data})
    response = request_to_gpt(messages)
    return response["choices"][0]["message"]["content"]
```

- Call function to analyze data in main
```py
if __name__ == '__main__':
    openai.api_key = os.getenv('OPENAI_API_KEY')
    data = # Read your data as string list

    result = []
    for d in data:
        result.append(analyze_data(d))

    with open('response.txt', 'w+') as f:
        f.write('\n'.join(results)+'\n')
    f.close()
```

## Asynchronous method to speed up API calls

The above solution can make API call automatically, even if reach the limitation of the API.<br>
But image you have a lot of data, for example, 100k.<br>
If there are 100k data need to communicate with GPT, it will take a really long time, maybe about 4 days.<br>
So, to avoid waiting for a long time, you can use asynchronous method to speed up API calls.<br>
And also, you can join two organizations.<br>

- Import multiprocessing.Pool to make asynchronous API call, and tqdm to display a processing bar for tracking
```py
from tqdm import tqdm
from multiprocessing import Pool
```

- Modify request_to_api function to allow specific organization
```py
@retry(wait=wait_random_exponential(min=30, max=60), stop=stop_after_attempt(5))
def request_to_gpt(messages: list, org: str):
    openai.organization = org
    return openai.ChatCompletion.create(
          model="gpt-3.5-turbo",
          messages=messages
    )
```

- Modify analyze_data parameter that contains organization, data and also index
```py
def analyze_data(data: tuple):
    review_text, org, index = data
    messages = []
    system_message = "Your QUESTION"
    messages.append({"role":"system","content":system_message})
    messages.append({"role":"user","content": review_text})
    response = request_to_gpt(messages, org)
    return str(index) + ": " + response["choices"][0]["message"]["content"]
```

- Define a function to read data and build input list
```py
def generate_data():
    org1 = os.getenv('OPENAI_API_ORG1')
    org2 = os.getenv('OPENAI_API_ORG2')

    data = # READ YOUR DATA

    org_list = [""] * len(data)
    index_list = [i for i in range(len(data))]
    for i in range(len(data)):
        if i%2 == 0:
            org_list[i] = org2
        else:
            org_list[i] = org1
    return list(zip(data, org_list, index_list))
```

- Create multiprocessing.Pool for API call
```py
if __name__ == '__main__':
    openai.api_key = os.getenv('OPENAI_API_KEY')
    data = generate_data()

    pool = Pool(4)
    results = list(tqdm(pool.imap(analyze_data, data), total=len(data)))

    with open('keywords.txt', 'w+') as f:
        f.write('\n'.join(results)+'\n')
    f.close()

    pool.close()
    pool.join()
```

## Notes
  - The result will be ordered the same as the input
  - The thread will execute until the task finish, if the task is waiting for retry, it will also wait. So maybe it is better to call it parallel method.
  - Make sure run this script in .py and run by command line, not in .ipynb. Seems Pool is not available in jupyter.
