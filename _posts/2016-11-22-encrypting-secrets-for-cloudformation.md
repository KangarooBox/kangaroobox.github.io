---
layout: post
title: Encrypting Secrets For CloudFormation
author: Richard Hurt
tags: cloudformation custom lambda aws secrets encryption
comments: true
---

One of the great things about CloudFormation is the ability to store all of your configurations in Git (or some other version control software).  The downside is that several CFN (CloudFormation) resources require secret information which then gets stored in your VCS as well.  Things like creating RDS instances or IAM users require you to provide cleartext passwords in your template.  But I've found a way around that using Lambda and KMS to decrypt your secrets when the CFN stack is Created/Updated.

First, I created a [Lambda function](https://gist.github.com/rnhurt/67a32139ca03030741876be5d009fb9a) that accepts a [CFN Custom resource](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html).  Below is a snippet of the Lambda Javascript showing the decryption of ciphertext using KMS:

```javascript
...    
    let responseStatus = 'FAILED';
    let responseData = {};
    var AWS = require('aws-sdk');
    var KMS = new AWS.KMS({apiVersion:'2014-11-01', region:event.ResourceProperties.Region});
    var params = {
        CiphertextBlob: Buffer(event.ResourceProperties.CiphertextBlob, 'base64'),
        EncryptionContext: event.ResourceProperties.EncryptionContext
      }; 

    KMS.decrypt(params, function(err,data){
      if(err){
        responseData = { Error: 'Unable to decrypt blob: '+err};
            console.log(`${responseData.Error}:\n`, err);
        } else {
            responseStatus = 'SUCCESS';
            // Name the values for the Fn::GetAtt CFN call.
            responseData = {Plaintext: data.Plaintext.toString()};
        }
        sendResponse(event, callback, context.logStreamName, responseStatus, responseData);
      }
    );
};
```
When CFN calls the custom resource it passes in an AWS Region, some ciphertext blob, and a context.  This Lamba processes the request, decrypts the ciphertext blob, and returns a plaintext string.

In the CFN template, you just have to use the correct Type (Custom::EncryptedValue) and fill in some Properties.  In order to get the plaintext value just use the "Fn::GetAtt" CFN function and name the attribute you want (i.e. "Plaintext").

```json
"iamp{{this.Name}}":{
  "Type":"Custom::EncryptedValue",
  "Properties":{
    "ServiceToken":"arn:aws:lambda:us-east-1:111222333:function:cfnEncryptedValue",
    "Region": { "Ref": "AWS::Region" },
    "CiphertextBlob":"{{this.Password}}",
    "EncryptionContext":{"Service":"IAM"}
  }
},

"iamu{{this.Name}}" : {
  "Type" : "AWS::IAM::User",
  "Properties" : {
    "Password": { "Fn::GetAtt": ["iamp{{this.Name}}","Plaintext"] },
    "Path" : "/{{this.Path}}",
    "UserName": "{{this.Name}}"
  }
}
```
_NOTE: I'm using the [CFN-Builder](https://github.com/KangarooBox/cfn-builder) engine to create these templates, which uses [Handlebars](http://handlebarsjs.com/) to create these CFN templates._ 