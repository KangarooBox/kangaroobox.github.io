---
layout: post
title: Counting Deployments in AWS
author: Richard Hurt
tags: cloud codedeploy aws
comments: true
---

I recently needed to know how many deployments we had in our CodeDeploy environment.  This is an easy enough process if you have a small(er) number of deployments, but once you get more than 100 it involves issuing multiple requests to the AWS CLI.  This involves using a token to get the next batch of deployments which is not super easy from the CLI.  So, I wrote a stupid little BASH script to do it for me.  This script could easily be modified to do other things with the AWS CLI that involves pagination (ie. nextToken).  :)

```bash
#!/usr/bin/env bash
status=${1:-'Created Queued InProgress Succeeded Failed Stopped'}

i=$(aws deploy list-deployments --include-only-statuses $status)
nextToken=$(echo $i | jq -r '.nextToken')
counter=$(echo $i | jq '.deployments[]' | wc -l)

while [ $nextToken ]
do
  i=$(aws deploy list-deployments --next-token $nextToken --include-only-statuses $status)
  nextToken=$(echo $i | jq -r '.nextToken')
  count=$(echo $i | jq '.deployments[]' | wc -l)
  counter=$((counter + count))

  echo "counter: $counter"
done

echo "Found $counter deployments"
```

One thing to note here is that this script uses [JQ](https://stedolan.github.io/jq/) (the excellent CLI JSON parser) to grab the nextToken information from the AWS return JSON.  If you don't have it installed or you don't want to use it for some reason, you should be able to modify the script to return just want you want using the AWS [```output```](http://docs.aws.amazon.com/cli/latest/userguide/controlling-output.html#text-output) and [```query```](http://docs.aws.amazon.com/cli/latest/userguide/controlling-output.html#controlling-output-filter) options.