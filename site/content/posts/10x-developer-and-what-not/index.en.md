---
title: "10x Developer and What not"
date: 2022-08-13T21:57:40+08:00
lastmod: 2022-08-14T16:45:40+08:00
description: "This article shows how to use Firecracker to create cluster VMs with container."
resources:
- name: "featured-image"
  src: "featured-image.webp"

tags: ["Design Choice", "Coding", "10x developer", "Business Impace"]
categories: ["Thought"]

---

What seperate between 10x developer and 1x developer, the line have to be really big to satisfy the term 10x developer. Most people think knowlege and experience is what cause the difference but every company have their own problem. The knowlege require between these jobs can be so difference that sometime you have to start from zero. For example mostly stack in Google is build in-house and they really do not care what stack and framework you've learn in the pass. The impactness and productivity of your performance is mostly value and solid metric to compare. But personal productivity always depends on a context and how can someone be 10x productivy than the other even within the same timeframe to solve the task is the same. The question is: do such super-productive professionals even exist, and is it possible to become one of them?

<!--more-->

Last week, I just watched the video of how one engineer can get 4 promotions in just one year and some ideas from it is very interesting and is the main reason this blog exist. I encourage people should check out this interview before reading the blog for more insight

{{< youtube 3_Ue0tweDkE >}}

## Some ideas from the interview video

I will summary some ideas I got from the interview. The impactness is the most concrete KPI for one to leverage between promotions, here is the example.
When the company reach the certain scale that need to optmize the cost in cloud vendor. First intution came to most developer is optmize coding, use cheaper cloud vendor, research service cloud offer and tuning configurations. But the engineer that was mention in the video have a very difference approach. Rather than dive deep in technology that will cost multiple months to analyst and deploy, he analyst the traffic and investigate the cloud cost between all users. Find out there are so many users that use the service is not the right way and even cost the company money rather than generate outcome. When he done analyst and list out some users cost the most, he reach out to other departments to alert them and brainstorm to optimize cost. After few weeks, he succeed to optimize the cloud cost nearly the target KPI only by his own.

{{< admonition success >}}

So rather than spend months to optimize cost via coding or cloud configuration that will cause the risk of downtime or bug, he only needs few weeks to achive the KPI in the most effective way.

{{< /admonition >}}

## My two cent

Similar to the interview, I had a recently problem that want to share that maybe you can find exciting. My current company is Jobhopin that offer Sas solution for parsing and ATS system. Mainly idea of our service is to automation process of input candidate info and auto matching the user's talent pool to their available jobs.

We had a problem that our platform resource does not meet the demand from customer, request die very frequently and of course the customer do not happy about it. So I was assigned tasks to optimize the platform for better user experience. And my fist thought maybe is everyone first thought, analyst the code and scaling technique to check for improvement. But I step back and rethought about the constraint of analyst engineer aspect. What if I done optimize cost, platform and then what, how much I can squeeze out from the code that enough to achive KPI, can my optimize code is bug-free and if not can I deleiver the code that won't break the existing platform ...etc. The constraint in engineer aspect is so huge but the reward is not clearly foreseen. That's why rather than continue dive more in coding, I thought - why our platform fail to deliver, what user expected from our platform and how much traffic that we are talk about. After a whole day debuging network and analyst log, I saw the abonormal trend in our user traffic that cause our platform can not scale quickly enough to handle traffic. The reality is each user have a very difference traffic, sometime they don't send any requests for the whole week but send few thousand requests in only one hours.

Turn out user rarely use our service but only in their recruitment event, therefore each event generate thousand of new candidates to our platform for analyst in only short span time that cause our service can scale well. And the other case is newly sign user, they migrate their talent pools to our service that cause dozen thousand traffics and rarely had any new candidate afterward. After saw these abnomarl trend, I propose to migrate our service from API Protocol to Queue System that can break out to two queues:

- Short queue (1 -> 50 candidates) using Lambda to serve
- Long queue (50> candidates) using ECS-Fargate task to serve

After replace to new protocol, the service handle well with the abonormal traffic from our user and even more, our cloud cost is half of what we paying before because we only pay what what user use, no more cost for idle server.

The process I descibe maybe is too vivid and how can someone learn to be one. Actually this process have a special term and have some books talk about it `Second-Order thinking`.

## Second-Order thinking

First-order thinking is fast and easy. Everybody can come up with their fist intution which is simplistic and superficial by nature. First-order thinking does not consider the negative ramifications of a decision in the future. For example, you can think of this as I’m hungry so let’s eat a chocolate bar.

{{< admonition quote >}}

Failing to consider second- and third-order consequences is the cause of a lot of painfully bad decisions, and it is especially deadly when the first inferior option confirms your own biases. Never seize on the first available option, no matter how good it seems, before you’ve asked questions and explored.”

—Ray Dalio

{{< /admonition >}}

Second-order thinking is a great mental model to have and helps us address some of the cognitive biases that get in our way when making choices. There are multiple ways to apply in practice that we can follow for better thinking

{{< admonition quote >}}

- Always ask yourself “And then what?”

- Think through time — What do the consequences look like in 10 minutes? 10 months? 10 Years?

- Create templates like the second image above with 1st, 2nd, and 3rd order consequences. Identify your decision and think through and write down the consequences. If you review these regularly you’ll be able to help calibrate your thinking.

- Learn to utilize feedback loops to help you make better decisions. In other words, will the decision you make give you accurate and timely feedback on its effectiveness? Here, it’s important to realize that the power of good decision making compounds over time – so continue to rework and refine your processes.


- (Bonus) If you’re using this to think about business decisions, ask yourself how important parts of the ecosystem are likely to respond. How will employees deal with this? What will my competitors likely do? What about my suppliers? What about the regulators? Often the answer will be little to no impact, but you want to understand the immediate and second-order consequences before you make the decision.

—[fs.blog](https://fs.blog/second-order-thinking/)

{{< /admonition >}}

[Applied Economics: Thinking Beyond Stage One by economist Thomas Sowell](https://www.amazon.com/Applied-Economics-Thinking-Beyond-Stage/dp/0465003451) explores this in good detail, applying it to political and economic policies, in his book. You should definitely check out the book for more insight and it is really great read.

Thinking is a process rather than intution and it take a lot of works to finalize process and be quick in second-order of thinking. However, doing so is a smart way to separate yourself from the masses.
