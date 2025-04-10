---
title: "10x Developer and Second-Order thinking"
date: 2022-08-13T21:57:40+08:00
lastmod: 2022-08-14T16:45:40+08:00
description: "This article shows how to be 10x developer and Second Order thinking."
resources:
- name: "featured-image"
  src: "featured-image.webp"

tags: ["Design Choice", "Second-Order thinking", "10x developer", "Productivity", "Business Impact"]
categories: ["Thought"]

---

What separates between 10x developer and 1x developer, the line has to be really big to satisfy the term 10x developer. Most people think knowledge and experience cause the difference, but every company has its problem. The knowledge required for these jobs can be so different that sometimes you have to start from zero. Rather than knowledge and experience, impacts and productivity of your performance are mostly value and solid metrics to compare. But personal productivity always depends on context; how can someone be 10x more productive than others even within the same timeframe to solve the problem? The question is: do such super-productive professionals even exist, and is it possible to become one of them?

<!--more-->

Last week, I watched a video about how one engineer can get four promotions in just one year. Some ideas from it are fascinating and are the main reason this blog exists. I encourage people should check out this interview before reading the blog for more insight.

{{< youtube 3_Ue0tweDkE >}}

## Some ideas from the video

I will summarize some ideas I got from the interview. The impact of your outcome is the most concrete KPI for one to leverage between promotions; here is an example.

When the company reaches a certain scale, it needs to optimize the cost with a cloud vendor. Most developers' first intuition is to optimize coding, use cheaper cloud vendors, research service cloud offerings, and tune configurations. But the engineer mentioned in the video had a very different approach. Rather than dive deep into technology that would take multiple months to analyze and deploy, he monitored the traffic and investigated the cloud cost across all users. He found that many users were using the service incorrectly and were even costing the company money rather than generating revenue. When he was done with the analysis and listed out which users cost the most, he reached out to other departments to alert them and brainstorm ways to optimize cost. After a few weeks, he successfully reduced cloud costs to nearly the target KPI on his own.

{{< admonition success >}}

So rather than spend months optimizing costs via coding or cloud configuration that will cause the risk of downtime or bug, he only needs a few weeks to achieve the KPI in the most effective way.

{{< /admonition >}}

## My two cents

Similar to the interview, recently, I had a problem that I want to share that maybe you can find exciting. My current company offers a SAS solution for parsing and ATS system. Our service mainly aims to automate the process of inputting candidate info and scoring the user's talent pool to their available jobs.

We had a problem where our platform resources did not meet the demand from customers. Requests failed very frequently, and of course, the customers were not happy about it. So I was assigned to optimize the platform for a better user experience. My first thought—maybe everyone's first thought—was to read the code, look into scaling techniques, and find room for improvement. But I had some doubts and rethought the constraints from the engineering perspective. What if I had done all the fancy things to optimize the code and platform, but then what next? How much could I squeeze out of the code to achieve the KPI? Could my optimized code be bug-free? If not, could I deliver code that wouldn't break the existing platform, etc.? The constraints from the engineering side are enormous, but the reward is not guaranteed. That's why, rather than continuing to dive deeper into coding, I thought about why our platform failed to deliver, what users expected from it, and how much traffic we were actually dealing with. After a whole day of debugging the network and examining the log output, I noticed an abnormal trend in our user traffic that caused the platform to be unable to scale quickly enough to handle it. The reality was that each user had very different traffic patterns; sometimes, they wouldn't send any requests for a whole week, but then would send a few thousand requests in just five minutes.

It turns out users rarely use our service but only when they open their recruitment events; therefore, each event generates thousands of new candidates to our platform for parsing in only a short period and obviously our API protocol struggle to keep up. The other case is newly signed users; they migrate their talent pools to our service and cause a dozen thousands of new traffic but rarely have any new candidates afterward. After seeing these unusual trends, I propose to migrate our service from API Protocol to Queue System that can break out into two queues:

- Short queue (1 -> 50 candidates) using Lambda to serve
- Long queue (50> candidates) using ECS-Fargate to serve

{{< admonition success >}}

Replacing to new protocol gave us a huge benefit both in performance and cost. The service handle very well our users' unusual traffic thanks to serverless autoscaling. Even more, our cloud cost is half of what we are paying before because we only pay for what users use, with no more cost for an idle server.

{{< /admonition >}}

The process I describe may be too vivid and complicated for anyone to mimic and apply in practice. Actually this process has a particular scientific term, and some great books talk about it `Second-Order thinking`.

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

[Applied Economics: Thinking Beyond Stage One by economist Thomas Sowell](https://www.amazon.com/Applied-Economics-Thinking-Beyond-Stage/dp/0465003451) explores this in good detail, applying it to political and economic policies. You should check out his book for more insight and better expl
ltions.

Thinking is a process rather than intuition; it takes a lot of work to finalize the process and be quick in second-order thinking. However, doing so is a smart way to separate yourself from the masses.
