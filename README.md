# Serverless EBS Volume Tag Enforcer | Penny Pincher
Creating `EBS Volumes` is easy, But maintaining them is hard. Especially when there are no `Tags` to describe their purpose. To make our jobs easier, we will automate the clean up action with AWS Lambda Functions.


Our Boto Script will do the following actions and Delete the ones without any tags or the explicit exclude tags
1. Scan for `EBS Volumes` in `Available` State
1. Exclude the Volumes with the `Pre-Approved` Tags
1. Delete all other EBS Volumes which are,
   1. Without any `Tags`
   1. With `Tags` that are not 'Pre-Approved`

![Fig : (https://raw.githubusercontent.com/miztiik/serverless-ebs-penny-pincher/master/images/serverless-ebs-penny-pincher.jpg)

## Pre-Requisities
We will need the following pre-requisites to successfully complete this activity,
- Few EBS Volume(s) in `Available` State:
  - _At least_ one Volume having Tag "Key:Name" "Value:Do-Not-Delete"
  - Another Volume without any tags 
- IAM Role - _i.e_ `Lambda Service Role` - _with_ `EC2FullAccess` _permissions_

_The image above shows the execution order, that should not be confused with the numbering of steps given here_

## Step 1 - Lambda Penny Pincher Code
The below script is written in `Python 2.7`. Remember to choose the same in AWS Lambda Functions.

_Change the global variables at the top of the script to suit your needs._
```py
import boto3

# Set the global variables
globalVars  = {}
globalVars['Owner']                 = "Miztiik"
globalVars['Environment']           = "Test"
globalVars['REGION_NAME']           = "ap-south-1"
globalVars['tagName']               = "Serverless-EBS-Penny-Pincher"
globalVars['findNeedle']            = "Name"
globalVars['tagsToExclude']         = "Do-Not-Delete"

ec2       = boto3.resource('ec2', region_name = globalVars['REGION_NAME'] )

def lambda_handler(event, context):

    deletedVolumes=[]

    # Get all the volumes in the region
    for vol in ec2.volumes.all():
        if  vol.state=='available':

            # Check for Tags
            if vol.tags is None:
                vid=vol.id
                v=ec2.Volume(vol.id)
                v.delete()

                deletedVolumes.append({'VolumeId': vol.id,'Status':'Delete Initiated'})
                print "Deleted Volume: {0} for not having Tags".format( vid )

                continue

            # Find Value for Tag with Key as "Name"
            for tag in vol.tags:
                if tag['Key'] == globalVars['findNeedle']:
                  value=tag['Value']
                  if value != globalVars['tagsToExclude'] and vol.state == 'available' :
                    vid = vol.id
                    v = ec2.Volume(vol.id)
                    v.delete()
                    deletedVolumes.append( {'VolumeId': vol.id,'Status':'Delete Initiated'} )
                    print "Deleted Volume: {0} for not having Tags".format( vid )

    # If no Volumes are deleted, to return consistent json output
    if not deletedVolumes:
        deletedVolumes.append({'VolumeId':None,'Status':None})

    # Return the list of status of the snapshots triggered by lambda as list
    return deletedVolumes

```

## Step 2 - Configure Lambda Triggers
We are going to use Cloudwatch Scheduled Events to take backup everyday.
```
rate(1 minute)
or
rate(5 minutes)
or
rate(1 day)
# The below example creates a rule that is triggered every day at 12:00pm UTC.
cron(0 12 * * ? *)
```
_If you want to learn more about the above Scheduled expressions,_ Ref: [CloudWatch - Schedule Expressions for Rules](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html#RateExpressions)

## Step 3 - Verify EBS Volumes in AWS Dashboard

## Customizations
You can use many of the lamdba configurations to customize it suit your needs,

- `Concurrency`:_Increase as necessary to manage all your instances_
- `Memory` & `Timeout`: _If you have a large number of instances, you want to increase the `Memory` & `Timeout`_
- `Security`: _Run your lambda inside your `VPC` for added security_
  - `CloudTrail` : _You can also enable `CloudTrail` for audit & governance_

