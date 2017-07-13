---
layout: post
title: "Tracing API Gateway requests through Lambda in AWS"
date: 2017-07-13
---

Searching through API Gateway logs and trying to map a request to an API can be very tricky. You end up trying to match up timestamps to trace the request through to it's Lambda invocation. 

There is a way to trace requests from API Gateway to lambda. You need to add some boilerplate code to your gateway to pass an api request id to the Lambda but the change is trivial. It then becomes as simple as logging the api request in the lambda to be able to trace the call from the API Gateway entry point.

## Update API Gateway

1. Open up your Api Gateway
1. For each of your methods go to `Intergration Request > Body Mapping Templates > application/json`
1. Update your template to add the following json:
```
 "context" : {
      "request-id": "$context.requestId",
       ...
  }`
```
[See Context Variable Reference](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#context-variable-reference) for other variables you can pass.

## Update your lambda (python)

Now that you're passing the request context to lambda you can start logging the context to cloud watch. 

1. Open up each lambda function you want to change.
1. import the logging package `import logging`
1. Here is a basic static logging class in python for Lambda. You simply pass your message and the context object from the event.
```
class Logger:
    """
    Wrapper class around logging.

    Formats all log messages with Api Gateway Request Id and Requester Id.
    """
    logger = logging.getLogger()
    logger.setLevel(logging.DEBUG)

    @classmethod
    def debug(cls, msg, context):
        cls.logger.debug(cls._format(msg, context))

    @classmethod
    def error(cls, msg, context):
        cls.logger.error(cls._format(msg, context))

    @staticmethod
    def _format(msg, context):
        return '[APIG: {}] {}'.format(context['request-id'], msg)
```

Using the `Logger` class:

```
def my_handler(event, context):
	Logger.debug(json.dumps(event), event['context'])
```

## Pulling it all together

Now you can search through cloud watch for an API Gateway request and using the api request id trace what happened in the lambda logs. 

There is also an easier way.

## Using `apilogs`

Over on [github](https://github.com/rpgreen/apilogs) there is an excellent python project, apilogs, that eases the process of searching for and tracing requests in cloudwatch.

```
pip install apilogs
``` 

then 

```
 apilogs get --api-id xyz123 --stage test2 --profile myprofile --aws-region us-east-1 --start='2h ago' --end='1h ago' | grep "e0354bfa-de1c-4858-ac09-615b96492892"
```

The above line greps for all log messages that contain the api request id `e0354bfa-de1c-4858-ac09-615b96492892`. The output is nicely colour coded and easy to read.