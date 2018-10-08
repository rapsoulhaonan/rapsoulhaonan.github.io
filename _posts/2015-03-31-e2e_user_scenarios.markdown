---
title: "Using Slack to display the status of CodeDeploy"
subtitle: "Auto is better than not"
tags: [DevOps, AWS]
---
Slack是目前很多人工作中必不可少的通讯工具；CodeDeploy是AWS上方便部署代码的有力助手。每次部署代码时，你是否苦恼在运维组的频道中重复同样的话？现在结合这两个工具，当CodeDeploy状态发生变化时，让Slack自动发送消息。
<!--more-->

##设置Slack

首先到Manage找到Custom Integrations，创建Incoming WebHooks。一般来说，你可以直接访问`https://your-domin-name-goes-here.slack.com/services/new`。根据提示简单设置之后，你就会获得一个Webhook URL，之后会用到。Slack就设置好了。

##创建Lambda

这里Lambda指的是AWS里面的Lambda Function：

> AWS Lambda runs your code on a high-availability compute infrastructure and performs all the administration of the compute resources, including server and operating system maintenance, capacity provisioning and automatic scaling, code and security patch deployment, and code monitoring and logging. All you need to do is supply your Node.js code.

在AWS中找到Lambda Function模块，创建时`Code template`选None。粘贴下面这段代码，`注意用你自己的webhook url`。这是Node.js，你可以根据你的需要，很简单的修改消息的样式和内容。

{% highlight javascript %}
var https = require('https');
var util = require('util');

exports.handler = function(event, context) {
    console.log(JSON.stringify(event, null, 2));
    console.log('From SNS:', event.Records[0].Sns.Message);

    var postData = {
        "text": "*" + event.Records[0].Sns.Subject + "*",
    };

    var message = event.Records[0].Sns.Message;

    //path换上你自己的webhook url
    var options = {
        method: 'POST',
        hostname: 'hooks.slack.com',
        port: 443,
        path: '/services/your-slack-webhook-url-info-goes-here'
    };

    var req = https.request(options, function(res) {
      res.setEncoding('utf8');
      res.on('data', function (chunk) {
        context.done(null);
      });
    });
    
    req.on('error', function(e) {
      console.log('problem with request: ' + e.message);
    });    

    req.write(util.format("%j", postData));
    req.end();
};
{% endhighlight %}

然后就可以测试了。在Sample Event里面，粘贴下面简单的SNS信息：

{% highlight javascript %}
{
    "Records": [
        {
            "EventSource": "aws:sns",
            "EventVersion": "1.0",
            "EventSubscriptionArn": "arn:aws:sns:us-east-1:963529778735:deployment-notice:1b5f9fec-1628-4124-8caf-eb1d68c7c8dc",
            "Sns": {
                "Type": "Notification",
                "MessageId": "fc94858f-129b-546f-83c6-d5feec49dbdb",
                "TopicArn": "arn:aws:sns:us-east-1:963529778735:deployment-notice",
                "Subject": "CREATED: AWS CodeDeploy d-OJSFEI0CI in us-east-1 to Portal",
                "Message": "{\"region\":\"us-east-1\",\"accountId\":\"963529778735\",\"eventTriggerName\":\"Deployment Status Update\",\"applicationName\":\"Portal\",\"deploymentId\":\"d-OJSFEI0CI\",\"deploymentGroupName\":\"Acceptance\",\"createTime\":\"Fri Oct 07 19:50:41 UTC 2016\",\"completeTime\":null,\"status\":\"CREATED\"}",
                "Timestamp": "2016-10-07T19:50:42.170Z",
                "SignatureVersion": "1",
                "Signature": "TQCHNq0NrX5ldDTF11gBRz95HWkcCJ5fKOvKmv0klGB9A4rC5g1B4Ydv8Z8+DVJO4UszsiDcxnOQ9yHkkaNUZJZ9jLrEtoPtFINokZTJFVkM1kvhqYyOA2ex5zzv/Vs52EwO9aG80NPIS0b8haEHRYMh52DVn4J05QrzwiiHbit67d75ZvEWMCJKWJewGP9nPY4RwnJJnlalyCtmkubNaAJmhiNcqsTg/6TqgMsBXfYxvBVICzmNCtuQrbVRoXTDoCgtwx2G9mQnmM1VgH37MBBibMmc2D/EJpDg+sUico/KEG/s9x1BixDk5Xc6teUPc78kgfJ3enPUjbYEuQwBgQ==",
                "SigningCertUrl": "https://sns.us-east-1.amazonaws.com/SimpleNotificationService-b95095beb82e8f6a046b3aafc7f4149a.pem",
                "UnsubscribeUrl": "https://sns.us-east-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:us-east-1:963529778735:deployment-notice:1b5f9fec-1628-4124-8caf-eb1d68c7c8dc",
                "MessageAttributes": {}
            }
        }
    ]
}
{% endhighlight %}

点测试就可以在Slack里面看到消息了。

##Simple Notification Service

在AWS中找到SNS，新建一个Topic。然后设置这个Topic，创建一个subscription。协议选aws lambda，这个时候你就可以看到你刚刚创建的那个lambda了。这样，只要有消息传到SNS，它就会自动执行这个lambda。

##CodeDeploy

找到你的Deployment Group，创建trigger练到SNS中你刚刚创建的Topic就行了。这里我碰到了一个权限问题，解决方式在这篇文章里面：[Grant Amazon SNS Permissions to an AWS CodeDeploy Service Role](http://docs.aws.amazon.com/codedeploy/latest/userguide/how-to-sns-service-role-permission.html)

##最后

`rake deploy:development`...搞定！酷不酷？
