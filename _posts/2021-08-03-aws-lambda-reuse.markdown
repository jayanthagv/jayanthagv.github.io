---
layout: post
title:  "How often does AWS reuse lambda containers?"
date:   2021-08-03 13:30:00 +0530
categories: aws lambda
---
I first realised that AWS re-uses lambda containers when I created a lambda function, to fetch me screenshots of URLs I give it in the
request, using Node + Puppeteer. The function executions kept failing consistently after a few initial successful runs. I realized that there was a memory leak in my code. This would not be a problem if AWS freshly instantiated a new container each time.

The frequency of re-use of lambda containers matter if you're using any global variables. Popular global variables in lambda functions can be DB connections, headless browser instance, etc. These are known to be frequent sources of memory leaks.

To find out how often a container is re-used we would need a way to distinctly recognize a container. If a randomly generated identifier is assigned to a global variable outside of the lambda handler function it should persist when the lambda function is called again. It is of course possible that AWS just re-imports our code instead of recreating the container. But that would result in garbage collection and global memory being freed so it probably won't make a difference to the end user.

A python lambda function which accomplishes this would look something like this - 

{% highlight python %}
import random
import string
import time


instanceId = ''.join(random.choices(string.ascii_lowercase + string.digits, k=10))


def lambda_handler(event, context):
    time.sleep(0.5)  # sleep for 500ms to simulate processing a typical request
    return {
        'statusCode': 200,
        'body': instanceId
    }

{% endhighlight %}

Now we need something that would call this lambda function (through API gateway) and observe the `instanceIds`.
This python script will do the trick - 

{% highlight python %}
import json
import requests
import threading
import time


LAMBDA_URL = "ENTER_YOUR_AWS_API_GATEWAY_URL_HERE"
LOG_FILE = 'ENTER_A_TXT_FILE'
LAMBDA_USAGE_COUNTS = 'ENTER_A_JSON_FILE_NAME'


class LambdaTester(object):

    def __init__(self):
        self.lambdaInstancesCount = {}
        self.currentActiveRequests = 0
        self.totalRequests = 0
        self.threads = []

    def writeLog(self, msg):
        f = open(LOG_FILE, 'a')
        f.write('{}\n'.format(msg))
        f.close()

    def requestAboutToStart(self):
        self.currentActiveRequests += 1
        self.totalRequests += 1

    def requestFinished(self):
        self.currentActiveRequests -= 1

    def callLambda(self):
        self.requestAboutToStart()
        try:
            r = requests.get(LAMBDA_URL)
            instanceId = r.content.decode()
            self.lambdaInstancesCount[instanceId] = self.lambdaInstancesCount.get(instanceId, 0) + 1
        except:
            self.writeLog("Exception")
        self.requestFinished()

    def saveLambdaInstanceUsageCounts(self):
        f = open(LAMBDA_USAGE_COUNTS, 'w')
        json.dump(self.lambdaInstancesCount, f)
        f.close()

    def makeLambdaCalls(self, count):
        if count <= 0:
            return
        for threadNo in range(count):
            thread = threading.Thread(target=self.callLambda)
            thread.start()
            self.threads.append(thread)

    def concludeTest(self):
        for thread in self.threads:
            thread.join()
        self.writeLog("Total requests = {}".format(self.totalRequests))
        self.writeLog("Total lambda instances = {}".format(len(self.lambdaInstancesCount.keys())))
        self.saveLambdaInstanceUsageCounts()
        print("Test finished!!")

    def runTest(self):
        for i in range(600000):
            self.makeLambdaCalls(1)
            time.sleep(0.1)
        self.concludeTest()

    def runTestAsync(self):
        thread = threading.Thread(target=self.runTest)
        thread.start()
        return thread


if __name__ == '__main__':
    tester = LambdaTester()
    tester.runTest()

{% endhighlight %}

A server with a decent network traffic would probably recieve 10 requests per second, so the above script makes a lambda call every 100ms for about 1000 minutes/16.7 hours.

### Results
Total requests = 600000\
Total lambda instances = 72\
Mean of number of times a lambda container is re-used = 8333\
Median of number of times a lambda container is re-used = 10911

Since the container takes 500ms to respond it has the capacity to handle 2 requests per second.\
We made 10 requests per second, this would mean that 5 containers were running at any given point of time.\
Containers were destroyed and recreated 72/5 = 14.4 times over the course of this test.

=> The average container's lifetime was 1000/14.4 = 69.4 minutes/~1.15 hours
