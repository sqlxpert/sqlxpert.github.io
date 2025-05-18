---
layout: post
title: "Idempotence: Doing It More than Once"
author:
  - Paul Marcelin
image: /assets/images/electric-wires-marcelin-20250516.jpg
image_alt_text: Sets of 3 electrical wires, intersecting at the top of a light pole
---

_Basic Amazon Web Services commands handle repetition differently. This article suggests ways to assure consistent software designs and implementations across your own organization. You do not need to know programming or AWS._

---

## What Is Idempotence?

In organizations, just as in software, it is best to do work one time, and one time only.
Preventing repetition is difficult, and the results of repetition can be costly. In a modern software system, code might be triggered more than once because an [event](https://aws.amazon.com/event-driven-architecture/) is reported, or a [queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-queue-types.html) message is delivered, more than once. In a large or long-lived organization, people who know nothing of each other might solve the same problem more than once.

If we cannot eliminate repetition, we want to be sure that the result remains the same (or gets better, but never worse). This is my plain-language definition of [idempotence](https://en.wikipedia.org/wiki/Idempotence) .

_Compare adding 1 with multiplying by 1. Which operation is idempotent? In other words, which operation gives the same result no matter how many times we repeat it? Now, is adding 0 idempotent? Multiplying by 0? If you are not a programmer or an AWS user, feel free to skip any parts of the article that do not apply to you, and resume reading at [Five AWS Services, Compared](#five-aws-services-compared)._

## AWS Builds Incrementally

Amazon Web Services comprises hundreds of services, [launched at different times](https://en.m.wikipedia.org/wiki/Timeline_of_Amazon_Web_Services). This is [incrementalism](https://en.wikipedia.org/wiki/Incrementalism) : capabilities are added little by little. The advantage is innovation. A potential disadvantage is repeated effort that leads to different, and not necessarily better, results.

My tool for cutting AWS cloud computing costs, [github.com/sqlxpert/lights-off-aws](https://github.com/sqlxpert/lights-off-aws#lights-off) , stops computers and databases when they are not needed, and starts them again later. It can delete other kinds of expensive resources, and create them again later. It can also take backups on a schedule.
These operations _ought to be idempotent_. Trying to start a computer that is already running should not cause an error. Trying to back up a database when a backup was started just a few minutes ago should not cause an error, let alone produce another backup.

Lights Off uses five AWS services. As you will see, some basic commands in core AWS services are non-idempotent, and this suggests that AWS's work processes are non-idempotent. It could be an example of [Conway's Law](https://en.m.wikipedia.org/wiki/Conway%27s_law) : software matches the organization that produces it.

## AWS Services Approach Idempotence Differently

|AWS Service|Launched|Commands|Idempotence<br/>Mechanism|Exception Name<br/>+ Error Code (if different)|Token Name<br/>+ Restrictions (if any)|
|:---|:---:|:---:|:---:|:---:|:---:|
|EC2||`StartInstances`<br/>`StopInstances`|Built in|||
|RDS|After EC2|`StartDBInstance`<br/>`StopDBInstance`|&cross; Not idempotent|`InvalidDBInstanceStateFault`<br/>`InvalidDBInstanceState`&empty;||
|Aurora|After RDS| `StartDBCluster`<br/>`StopDBCluster`|User checks an error message|`InvalidDBClusterStateFault`||
|CloudFormation|Before Aurora|`UpdateStack`|User sets a token||`ClientRequestToken`<br/>&le;128 letters, numbers and hyphens|
|AWS Backup|After the others|`StartBackupJob`|User sets a token||`IdempotencyToken`|

### 1. EC2

Elastic Compute Cloud is the oldest of the five services. Its `StartInstances` and `StopInstances` commands are idempotent. If I try to start a compute instance that is already running, my request succeeds. The dynamic response message tells me that the instance was already running at the exact time of my request, in case I need to know.

```json
"StartingInstances": [
    {
        "InstanceId": "[redacted]",
        "CurrentState": {
            "Code": 16,
            "Name": "running"
        },
        "PreviousState": {
            "Code": 16,
            "Name": "running"
        }
    }
]
```

### 2. RDS

Relational Database Service was built on EC2, but its `StartDBInstance` and `StopDBInstance` commands are non-idempotent. If I try to start a database that is already running, I get an error. The [exception](https://en.wikipedia.org/wiki/Exception_handling) is named `InvalidDBInstanceStateFault` but the error code is `InvalidDBInstanceState` &mdash; a bug waiting to happen! The only thing the long error message _doesn't_ tell me is that the database was already running (available) at the exact time of my request. I cannot decide whether to ignore the error (because my start command was indeed a harmless repeat) or take it seriously (in case the database was in a bad state and could not start).

```text
An error occurred (InvalidDBInstanceState) when calling the StartDBInstance
operation: Instance [redacted] cannot be started as it is not in one of the
following statuses: 'stopped, inaccessible-encryption-credentials-recoverable,
incompatible-network (only valid for non-SqlServer instances)'.
```

### 3. Aurora

This newer relational database service's `StartDBCluster` and `StopDBCluster` commands produce an error with a matching exception name and error code, `InvalidDBClusterStateFault` . More importantly, the dynamic error message tells me that the database was already running (available) at the exact time of my request. Safely ignoring the error achieves idempotence after the fact.

```text
An error occurred (InvalidDBClusterStateFault) when calling the StartDBCluster
operation: DbCluster [redacted] is in available state but expected it to be
one of stopped, inaccessible-encryption-credentials-recoverable.
```

### 4. CloudFormation

This service, which creates and deletes all kinds of resources, predates Aurora. Its `UpdateStack` command is idempotent if I add a fixed value (token) to my request. If all the details, including the token I've chosen, match, then repeated requests succeed and CloudFormation acts only on the first request.  `ClientRequestToken` is limited to 128 alphanumeric characters and hyphens. Lights Off runs every ten minutes, so I set the token to the start of the ten-minute interval. I have to remove the colon that separates hours from minutes. The time still conforms to the ISO 8601 standard, but it becomes a little harder for humans to decipher (`T15:10` becomes `T1510`, for example).

### 5. AWS Backup

This is the newest of the five services. Its `StartBackupJob` command follows the same approach as CloudFormation, but the token is named `IdempotencyToken` and no specific limits are placed on length or permitted characters.

### Five AWS Services, Compared

The five services all take different approaches to idempotence. Within AWS, the left hand did not know what the right hand was doing. EC2, the oldest of the five services, and AWS Backup, the newest, handle idempotence well.

### Why Do We Care about AWS Inconsistencies?

Lack of consistency is **expensive**.
I do not know whether _AWS_ spent more by having each service team proceed independently, or whether it earned more by bringing each service to market faster.
I do know that every _customer_ using multiple AWS services has to discover the inconsistencies (sometimes by trial-and-error, because not every detail is or can be documented), write extra code to work around them, and fix the extra bugs that result.

For a specific example, consider the major AWS command-line interface (CLI) update in 2020. Part of the work involved covering up disparate date and time representations used by different AWS services. In version 1, customers paid for the lack of consistency. Eventually, AWS paid, by [having version 2 of the CLI convert dates and times to an ISO 8601 standard format](https://docs.aws.amazon.com/cli/latest/userguide/cliv2-migration-changes.html#cliv2-migration-timestamp).

## How Can We Avoid Repeating this Pattern?

As frustrating as the lack of consistency is to me as a software engineer, it is even more frustrating to me as an MBA. I propose a three-part solution for our own organizations: 

### A. Document Standards Centrally and Succinctly

I am not talking about a Confluence page tree of stale documents, requests for comment (RFCs), and so on. Nobody will read that. Anyone who peeks will get nervous because one author after another is inevitably, and ominously, labeled "A former colleague".

The goal is **to promote awareness, not to cover every detail**. Fit the list of standards to one or two pages. Group the entries by topic. Limit each entry to one sentence. For example, of the acceptable approaches to idempotence covered in this article, you could choose between:

#### The EC2 Approach

* Idempotence: A repeated request succeeds and is ignored.

#### The Aurora Approach

* Idempotence: A repeated request fails with an error message that mentions the state at the time of the request.

#### The AWS Backup Approach

* Idempotence: A repeated request succeeds and is ignored if `IdempotencyToken` , an arbitrary string, remains the same.

_Which two of the three approaches above are semantically equivalent? Would any one of the three approaches be enough to make all of the commands listed earlier in the article idempotent? If not, what makes one command or pair of commands special?_

To clarify a one-sentence standard, link to a few lines of actual code in your codebase. If you must, link to an explanatory document. Ideally, it will be an _external_ document. There is no need to invent date and time formats, metadata schemes, representations of numbers and units of measurement, etc. Groups of experts have already done the work; rely on external standards whenever possible.

### B. Review Past Practices when Designing and Coding

A review of past practices belongs in every important design, and also in the description of an important pull request (proposed code).
Add this to the "definition of done". Allocate time in proportion to the cost of changing the system later. For example, spend extra time searching for past practices when you are designing or implementing a public application programming interface (API). As soon as customers and their code depend on your API, changing the semantics becomes practically impossible.

Do not let searching for past practices devolve into a perfunctory step, a mere checkbox in a template. If you found no past practice, mention where you looked and what you looked for. Documenting the _absence_ of information is as important as documenting information.

### C. Maintain the Standards

Even a short list of standards can quickly fall out-of-date. Assign two members of each team to be responsible. Give them time, and recurring tickets, to edit their team's entries every quarter.  This is real work, worthy of recognition. It is some of the most valuable work in the organization, because of the time and money it saves. The savings may accrue to the organization itself, or to the customers.

Obviously, any person shepherding an important design or pull request should add to the standards list when setting a precedent, and update it when improving on a past practice. That is worthy of recognition, too. It is not, however, a substitute for informed, periodic editing of the list of standards.

We might not always anticipate the consequences of our design and implementation decisions. That is okay. The goal is to **gradually build habits of awareness, responsibility, and reciprocity**. My effort saves other people time, and their efforts save me time. As an organization, we try hard to prevent repetitive work. If it does happen, our disciplined practices encourage consistent results and discourage regression.

_Thanks for reading! I will eventually publish answers to the questions embedded in the article, but if you are curious or have ideas to share right away, please get in touch._
