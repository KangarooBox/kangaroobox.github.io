---
layout: post
title: VPC Subnet Addressing
author: Richard Hurt
tags: aws vpc ip_address cidr subnet networking
comments: true
---

There are a lot of ways to build VPC subnets in AWS and most of them are wrong.  The most common problem I run across is when 
people build them like they do a standard datacenter - lot's of subnets with small CIDR blocks.  This causes 2 problems; confusion 
and lack of space.  The confusion part is pretty simple, the more subnets you have the more ways you have to find the label them
and sort through them when you need to find somewhere to host your resource.  Pretend you have a WWW subnet and a Database subnet; where
would you put a Memcache resource?  It's (almost) a database and it's used primarily by the webservers so it could really go into 
either one without too much trouble.  Why cause yourself that much grief?

![AWS VPC](/assets/images/320px-AWS-vpc.png "AWS VPC logo"){: .right}

The other problem with building lots of small subnets is that you run out of IP addresses.  A /24 block only gives you 254 addresses to
work with, which may seem like a lot but in the cloud there are lots of things that take up addresses that you don't really think about.
For example, each ELB has to have a presence in at least 2 subnets and other things (S3 endpoints, DNS, etc.) take up even more space.
Plus you need room to grow and shrink your instances without having to worry about not having enough IP addresses.  Amazon even states
that it's better to go larger (with your addressing) than you think you need.  You'll thank yourself later.

Here is my suggestion for how to layout your VPC subnets.  There are no hard and fast rules but I've used this before and it seems to
work pretty well.  Feel free to modify it for your specific needs, but remember to keep the future in mind and not build things too
small.

# CIDR Allocations

## Customer (/13)

At the top of the chain is the customer, or overall purpose, of this AWS world.  It could be that you only have one "customer" and
that is your base organization.   Or you could have multiple literal customers that each need their own world.  Either way, this 
CIDR block is a _virtual_ organization that is never realized in a physical form.  It merely helps to align your mental model of how
your AWS architecture is constructed.

## Account

Each environment (aka Production, Staging, QA, Development, etc.) should have it's own account.  This is important for many reasons,
including separation of responsibilities and reducing the blast radius of any problems.  When your testing environment sends out 30,000
invalid email addresses you'll be glad that your production SES domain is in a completely different AWS account.

## VPC (/18)

Most people create their VPCs using a /16 block and while I think that's a good idea it limits you to only using 2 regions (in this
architecture anyway).  I think it's better to split that up a bit and use a /18 instead which gives you the ability to split your 
VPCs across 4 regions.  If you are not thinking of using multiple regions (or just 2 regions) then feel free to modify this block to a
/16.

## Subnets

### Private (/20)

There are 3 major subnet types; private, public, & transit.  The private subnets are the largest @ /20 (\~4,000 addresses) and contain
most of your resources.  This is where you run your databases, web servers, application servers, etc. so it needs to be the largest.

### Public (/22)

The Public subnet is where you host things that ~have~ to be exposed to the raw Internet.  This includes things like load balancers and 
certain types of "other" services (i.e. FTP) that can't live behind an ELB.  Public subnets should be smaller than Private ones because
you really shouldn't be running anything in them unless absolutely necessary.

### Transit (/24)

This subnet type only came about because of the [Transit Gateway service](https://aws.amazon.com/transit-gateway/) launched at AWS re:Invent 2018.  It's main goal in life is to 
connect this VPC to other VPCs through the Transit Gateway service.  Strictly speaking, this subnet type is not required and you 
could do everything you need to without it, but it does make certain network operations cleaner and it doesn't use up to many 
addresses (\~250).


# Example

Suppose you have a new customer (ABC Corp) and you want to give them their own space to work in with the usual environments (Dev, QA,
Prod).  You start off by assigning them an unused /13 block from your list of available IP address blocks (you do keep track of the
CIDR blocks you've used, right?)  In this case, we'll use 10.200.0.0/13 which will give us 8 "environments" or accounts to work 
with (10.200.0.0 - 10.207.255.255).

Below you can see a couple of layouts for 2 different environments (production, development) each in 2 regions (us-east-1, us-west-2):

## Production Account
* **Account**
  * **Region** (us-east-1)
    * **VPC** - 10.200.0.0/18
      * **Subnets**
        * **Private**
          * subnet a - 10.200.0.0/20
          * subnet b - 10.200.16.0/20
          * subnet c - 10.200.32.0/20
        * **Public**
          * subnet a - 10.200.58.0/22
          * subnet b - 10.200.52.0/22
          * subnet c - 10.200.56.0/22
        * **Transit**
          * subnet a - 10.200.60.0/24
          * subnet b - 10.200.61.0/24
          * subnet c - 10.200.62.0/24

  * **Region** (us-west-2)
    * **VPC** - 10.200.64.0/18
      * **Subnets**
        * **Private**
          * subnet a - 10.200.64.0/20
          * subnet b - 10.200.80.0/20
          * subnet c - 10.200.96.0/20
        * **Public**
          * subnet a - 10.200.112.0/22
          * subnet b - 10.200.116.0/22
          * subnet c - 10.200.120.0/22
        * **Transit**
          * subnet a - 10.200.124.0/24
          * subnet b - 10.200.125.0/24
          * subnet c - 10.200.126.0/24

## Development Account
* **Account**
  * **Region** (us-east-1)
    * **VPC** - 10.201.0.0/18
      * **Subnets**
        * **Private**
          * subnet a - 10.201.0.0/20
          * subnet b - 10.201.16.0/20
          * subnet c - 10.201.32.0/20
        * **Public**
          * subnet a - 10.201.58.0/22
          * subnet b - 10.201.52.0/22
          * subnet c - 10.201.56.0/22
        * **Transit**
          * subnet a - 10.201.60.0/24
          * subnet b - 10.201.61.0/24
          * subnet c - 10.201.62.0/24

  * **Region** (us-west-2)
    * **VPC** - 10.201.64.0/18
      * **Subnets**
        * **Private**
          * subnet a - 10.201.64.0/20
          * subnet b - 10.201.80.0/20
          * subnet c - 10.201.96.0/20
        * **Public**
          * subnet a - 10.201.112.0/22
          * subnet b - 10.201.116.0/22
          * subnet c - 10.201.120.0/22
        * **Transit**
          * subnet a - 10.201.124.0/24
          * subnet b - 10.201.125.0/24
          * subnet c - 10.201.126.0/24
