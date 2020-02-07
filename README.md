# core-cloudwatch2humio
Humio’s CloudWatch integration sends your AWS CloudWatch Logs to Humio by using an AWS Lambda function to ship the data.

The integration is available from GitHub:

https://github.com/humio/cloudwatch2humio

# Launch Parameters
Humio installs the integration using a CloudFormation template.

The template supports the following parameters:

+ StackName - Globally unique stackname. Integration uses an S3 bucket with stackname as bucket name and bucket names need to be unique
+ HumioHost — The host you want to ship your Humio logs to.
+ HumioDataspaceName — The name of the repository in Humio that you want to ship logs to.
+ HumioAutoSubscription — Enable automatic subscription to new log groups.
+ HumioIngestToken — The value of your ingest token from your repository in your Humio account. (Ingest tokens can be found under Settings in each repository)
+ HumioSubscriptionBackfiller — This will check for missed or old log groups that existed before the Humio integration will install. This increases execution time of the lambda by about 1s. Defaults to true.
+ HumioProtocol — The transport protocol used for delivering log events to Humio. HTTPS is default and recommended, but HTTP is possible as well.
+ HumioSubscriptionPrefix — By adding this filter the Humio Ingester will only subscribe to log groups whose paths start with this prefix

# How Integration Works
The integration will install four lambda functions, the AutoSubscriber,CloudwatchIngester, the CloudwatchBackfiller and the LambdaInvoker. The CloudFormation template will also set up CloudTrail and an S3 bucket for your account for handling AWS event from CloudWatch when new loggroups are created. We need this to trigger the Auto Subscription lambda to newly created log groups. Finally an S3 bucket will be setup for our account for cold storage of logs. This is needed to keep logs more than 30 days which is the default retention period for logs in Humio.

## CloudwatchIngester
This lambda handles the delivery of your CloudWatch log events to Humio.

## AutoSubscriber
This lambda will auto subscribe the CloudwatchIngester every time a new log group is created. This is done by filtering CloudTrail events and triggering the AutoSubscriber lambda every time a new log group is created.

## CloudwatchBackfiller
This will run if you have set HumioSubscriptionBackfiller to true when executing the CloudFormation template. This function will paginate through your existing CloudWatch log groups and subscribe the CloudwatchIngester to every single one.

## LambdaInvoker
This will only run once after deploy of the other lambdas and trigger the AutoSubscriber so it will start listening for new log groups and trigger the CloudwatchBackfiller to subscribe to existing log groups

## Cold Storage
Logs are only kept queryable in Humio for 30 days but we need to keep them for longer time for BI purposes and if we need to look for errors older than 30 days. Therefore an S3 bucket is created to keep these logs for as long as we like. Archiving from Humio to S3 have to be setup manually in each Humio repository where you enter the name of the bucket you want Humio to send logs to, the region and the log format. As default we are using NDJSON format. 


# Implementation details
## Nuget Package: AWSSDK.Core
Base SDK for .net Core runtime

## Nuget Package: AWSSDK.CloudWatchLogs
Contains the class 'AmazonCloudWatchLogsClient' which gives access to CloudWatch groups, streams and subscription filters.

## Structure of CloudWatchEvent 
The following data is what a lambda app receives when no modification has been done to it however only **detail.requestParameters.logGroupName** is used.
```json
{
    "version": "0",
    "id": "0b49e3d7-ca84-ba00-2dd8-cca3e8d40016",
    "detail-type": "AWS API Call via CloudTrail",
    "source": "aws.logs",
    "account": "445130177860",
    "time": "2020-02-05T14:59:35Z",
    "region": "eu-central-1",
    "resources": [],
    "detail": {
        "eventVersion": "1.05",
        "userIdentity": {
            "type": "IAMUser",
            "principalId": "AIDAWPI6TPVCH7PHW2PLH",
            "arn": "arn:aws:iam::445130177860:user/jtk",
            "accountId": "445130177860",
            "accessKeyId": "ASIAWPI6TPVCERM5KVXF",
            "userName": "jtk",
            "sessionContext": {
                "sessionIssuer": {},
                "webIdFederationData": {},
                "attributes": {
                    "mfaAuthenticated": "false",
                    "creationDate": "2020-02-05T07:52:33Z"
                }
            },
            "invokedBy": "signin.amazonaws.com"
        },
        "eventTime": "2020-02-05T14:59:35Z",
        "eventSource": "logs.amazonaws.com",
        "eventName": "CreateLogGroup",
        "awsRegion": "eu-central-1",
        "sourceIPAddress": "217.198.216.74",
        "userAgent": "signin.amazonaws.com",
        "requestParameters": {
            "logGroupName": "test2"
        },
        "responseElements": null,
        "requestID": "c2ea907d-6c45-41d1-a57f-313bfa3cf9e0",
        "eventID": "db38ac56-778b-40c5-8b87-ab37fa640af8",
        "eventType": "AwsApiCall",
        "apiVersion": "20140328"
    }
}
```

## Structure of LogEvent sent from CloudWatch
The following data is what a lambda app receives. The data is encoded in base64 and afterwards compressed using gzip.
```json
{
    "awslogs": {
        "data": "H4sIAAAAAAAAADWQwW7DIBBEfyXiHOQFloBzsxQ3p56cWxVV2CYRkm0swI2qKP9e7LbHndW82Z0nGW2M5m4v37MlR3KqLtXne9001bkme+Ifkw1ZRpRMAFNKHyDLg7+fg1/mvCnMIxad7227uKEvTBfpHHy/dMn5ibbep5iCmX9NTQrWjNkFouWAt5Iy6BVFK0qqlQAqSmTcMGX7LScubeyCm1fWmxuSDZEcP0iyMd22kVw3bv1lp7SunsT1GS8klwIVyzQJUnHJJGqNmnENHBkC5kcOgjNAXvJDyYVWKABzZHK5kGTG/BuTGjQHISUA7P+LyvjGj3a3HrH70xgX5HV9/QB2nn7ITQEAAA=="
    }
}
```

The following data is the structure you get from decoding and decompressing the value of the data property from above.

```json
{ 
   "messageType":"DATA_MESSAGE",
   "owner":"445130177860",
   "logGroup":"/aws/codebuild/acs-production-bootstrap",
   "logStream":"03b204f9-10d7-4e39-8730-39412a17ed60",
   "subscriptionFilters":[ 
      "testfilter"
   ],
   "logEvents":[ 
      { 
         "id":"35253471941505725154884812802414047866321042926923874304",
         "timestamp":1580820355000,
         "message":"Some test message123"
      }
   ]
}
```

