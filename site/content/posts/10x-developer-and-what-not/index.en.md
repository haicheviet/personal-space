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

What separates between 10x developer and 1x developer, the line has to be really big to satisfy the term 10x developer. Most people think knowledge and experience cause the difference, but every company has its problem. The knowledge required for these jobs can be so different that sometimes you have to start from zero. Rather than knowledge and experience, impacts and productivity of your performance are mostly value and solid metrics to compare. But personal productivity always depends on context; how can someone be 10x more productive than others even within the same timeframe to solve the problem? The question is: do such super-productive professionals even exist, and is it possible to become one of them?

<!--more-->

Last week, I watched a video about how one engineer can get four promotions in just one year. Some ideas from it are fascinating and are the main reason this blog exists. I encourage people should check out this interview before reading the blog for more insight.

{{< youtube 3_Ue0tweDkE >}}

## Some ideas from the video

I will summarize some ideas I got from the interview. The impact of your outcome is the most concrete KPI for one to leverage between promotions; here is an example.

When the company reaches a certain scale, that needs to optimize the cost in a cloud vendor. Most developers' first intuition is to optimize coding, use cheaper cloud vendors, research service cloud offerings, and tuning configurations. But the engineer mentioned in the video has a very different approach. Rather than dive deep into technology that will cost multiple months to analyze and deploy, he analyst the traffic and investigate the cloud cost between all users. Find out that many users use the service incorrectly and even cost the company money rather than generate the outcome. When he is done with the analysis and lists out some users' costs the most, he reaches out to other departments to alert them and brainstorm to optimize cost. After a few weeks, he successfully reduced cloud cost to nearly the target KPI only on his own.

{{< admonition success >}}

So rather than spend months optimizing costs via coding or cloud configuration that will cause the risk of downtime or bug, he only needs a few weeks to achieve the KPI in the most effective way.

{{< /admonition >}}

## My two cent

Similar to the interview, recently, I had a problem that I want to share that maybe you can find existing. My current company, Jobhopin, offers a SAS solution for parsing and ATS system. Our service mainly aims to automate the process of inputting candidate info and auto-matching the user's talent pool to their available jobs.

We had a problem that our platform resource does not meet the demand from the customers, requests die very frequently, and of course, the customers do not feel happy about it. So I was assigned to optimize the platform for a better user experience. And my first thought, maybe is everyone first thought, analyze the code and scaling technique to check for improvement. But I stepped back and rethought the constraint of the engineering aspect. What if I had done optimized code or platform, and then what next? How much can I squeeze out from the code enough to achieve KPI, can my optimized code be bug-free, and if not, can I deliver the code that won't break the existing platform ...etc.? The engineering aspect's constraint is enormous, but the reward is not foreseen. That's why rather than continue diving more into coding, I thought about why our platform failed to deliver, what users expect from our platform and how much traffic we are talking about. After a whole day of debugging the network and analyst log, I saw the abnormal trend in our user traffic that caused our platform can not scale quickly enough to handle. The reality is each user has very different traffic; sometimes, they don't send any requests for the whole week but send a few thousand requests in only five minutes.

It turns out users rarely use our service but only in their recruitment events; therefore, each event generates thousands of new candidates to our platform for analysis in only a short period. Of course, our service stutter to keep up with the API protocol. And the other case is newly signed users; they migrate their talent pools to our service, cause a dozen thousand traffic, and rarely have any new candidates afterward. After seeing these abnormal trends, I propose to migrate our service from API Protocol to Queue System that can break out into two queues:

- Short queue (1 -> 50 candidates) using Lambda to serve
- Long queue (50> candidates) using ECS-Fargate to serve

{{< admonition success >}}

Replacing to the new protocol give us a huge benefit both in performance and cost. The service handle well with the abonormal traffic from our user thanks to servel and even more, our cloud cost is half of what we are paying before because we only pay what what user use, no more cost for idle server.

{{< /admonition >}}

The process I describe may be too vivid and complicated for anyone to mimic and apply in practice. But this process has a particular scientific term, and some great books talk about it `Second-Order thinking.`

## Second-Order thinking

First-order thinking is fast and easy. Everybody can come up with their first intuition which is simplistic and superficial by nature. First-order thinking does not consider the negative ramifications of a decision in the future. For example, you can think of this as I’m hungry so let’s eat a chocolate bar.

{{< admonition quote >}}

Failing to consider second- and third-order consequences is the cause of a lot of painfully bad decisions, and it is especially deadly when the first inferior option confirms your own biases. Never seize on the first available option, no matter how good it seems, before you’ve asked questions and explored.

—Ray Dalio

{{< /admonition >}}

Second-order thinking is a great mental model to have and helps us address some of the cognitive biases that get in our way when making choices. There are multiple ways to apply in practice that we can follow for better thinking.

{{< admonition quote >}}

- Always ask yourself “And then what?”

- Think through time — What do the consequences look like in 10 minutes? 10 months? 10 Years?

- Create templates like the second image above with 1st, 2nd, and 3rd order consequences. Identify your decision and think through and write down the consequences. If you review these regularly you’ll be able to help calibrate your thinking.

- Learn to utilize feedback loops to help you make better decisions. In other words, will the decision you make give you accurate and timely feedback on its effectiveness? Here, it’s important to realize that the power of good decision making compounds over time – so continue to rework and refine your processes.

- (Bonus) If you’re using this to think about business decisions, ask yourself how important parts of the ecosystem are likely to respond. How will employees deal with this? What will my competitors likely do? What about my suppliers? What about the regulators? Often the answer will be little to no impact, but you want to understand the immediate and second-order consequences before you make the decision.

—[fs.blog](https://fs.blog/second-order-thinking/)

{{< /admonition >}}

[Applied Economics: Thinking Beyond Stage One by economist Thomas Sowell](https://www.amazon.com/Applied-Economics-Thinking-Beyond-Stage/dp/0465003451) explores this in good detail, applying it to political and economic policies, in his book. You should check out the book for more insight, it is very great book for thought.

Thinking is a process rather than intuition; it takes a lot of work to finalize the process and be quick in second-order thinking. However, doing so is a smart way to separate yourself from the masses.