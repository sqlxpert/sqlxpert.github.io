# Doing It More than Once

Paul Marcelin, 2025-05-15

## What Is Idempotence?

In organizations, just as in software, it is best to do work one time, and one time only.
Preventing repetition is difficult, and the results of repetition can be costly. In a modern software system, a circumstance (an [event](https://aws.amazon.com/event-driven-architecture/)) that triggers an action might be reported more than one time, or a message in a [queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-queue-types.html) might be delivered for processing more than one time.
In a large or long-lived organization, different people who know nothing of each other might solve the same problem more than one time.

If we cannot eliminate repetition, we want to be sure that the result remains the same (or gets better, but never worse). This is my plain-language definition of [idempotence](https://en.wikipedia.org/wiki/Idempotence) .

Compare adding 1 with multiplying by 1. Which operation is idempotent? In other words, which operation gives the same result no matter how many times we repeat it? Now, is adding 0 idempotent? Multiplying by 0?

## AWS Builds Services Incrementally

My tool for cutting Amazon Web Services cloud computing costs, [github.com/sqlxpert/lights-off-aws](https://github.com/sqlxpert/lights-off-aws#lights-off) , stops computers and databases when they are not needed, and starts them again later. It can delete other kinds of expensive resources, and create them again later. It can also make backups on a schedule.
These operations _ought to be idempotent_. Trying to start a computer that is already running should not cause an error. Trying to back up a database when a backup was started a few minutes ago should not cause an error, let alone produce another backup.

AWS consists of hundreds of services. Lights Off uses five. AWS built the services at [different times](https://en.m.wikipedia.org/wiki/Timeline_of_Amazon_Web_Services). This is incrementalism: capabilities are added little by little. Its advantage is rapid innovation. One of its disadvantages is repeated effort that leads to different, and not necessarily better, results. As you will see, some basic commands in core AWS services are non-idempotent, and this suggests that AWS's internal work processes are non-idempotent. It could be an example of [Conway's Law](https://en.m.wikipedia.org/wiki/Conway%27s_law) : the software and the organization that produced it match each other.

## AWS Services Approach Idempotence Differently

|AWS Service|Introduced|Commands|Idempotence<br/>Mechanism|Error Name,<br/>Code (if different)|Token Name,<br/>Rules (if any)|
|:---|:---:|:---:|:---:|:---:|:---:|
|EC2||`StartInstances`<br/>`StopInstances`|Automatic|||
|RDS|After EC2|`StartDBInstance`<br/>`StopDBInstance`|None|`InvalidDBInstanceStateFault`<br/>`InvalidDBInstanceState`||
|Aurora|After RDS| `StartDBCluster`<br/>`StopDBCluster`|Error-handling|`InvalidDBClusterStateFault`||
|CloudFormation|Before Aurora|`UpdateStack`|Token||`ClientRequestToken`<br/>&le;128 letters, numbers, hyphens|
|AWS Backup|After the others|`StartBackupJob`|Token||`IdempotencyToken`|

1. Of the five services used, **EC2 (Elastic Compute Cloud)** came first. Its `StartInstances` and `StopInstances` commands are idempotent. If I try to start a compute instance that is already running, my request succeeds. (The response message kindly tells me that the instance was already running at the exact time of my request, in case I need to know.)

2. **RDS (Relational Database Service)** was built on EC2, but its `StartDBInstance` and `StopDBInstance` commands are non-idempotent. If I try to start a database that is already running, I get an error. The error is named `InvalidDBInstanceStateFault` but the error code is `InvalidDBInstanceState` . (This difference is a bug waiting to happen!) The error message doesn't tell me that the database was already running at the exact time of my request, so I cannot decide whether to ignore the error (because my start command was indeed a harmless repeat) or take it seriously (because the database was in a bad state and could not be started).

3. **Aurora**, a newer database service, is similar, but its `StartDBCluster` and `StopDBCluster` commands are better. First, the error they produce shares the same name and error code, `InvalidDBClusterStateFault` . Second, the error message tells me what state the database was in at the exact time of my request. I receive enough information to decide whether to ignore the error (achieving idempotence after the fact) or take it seriously.

4. **CloudFormation**, a service that creates and deletes all kinds of resources, and that predates Aurora, has an `UpdateStack` command that is optionally idempotent. I can specify a known value, a token. If all the details, including the token, match, then repeated requests succeed; CloudFormation acts on the first request only. The token is named `ClientRequestToken` , cannot contain punctuation other than hyphens, and cannot be more than 128 characters long. Lights Off runs every ten minutes, so I set the token to the start of the ten-minute interval. I have to remove the colon that separates hours from minutes. The time still conforms to the ISO 8601 standard, but it becomes a little harder for humans to decipher (for example, `1510` instead of `15:10`).

5. **AWS Backup** is the newest of the five services. Its `StartBackupJob` command follows the same approach as CloudFormation, but the token is named `IdempotencyToken` , can contain punctuation, and is not limited to 128 characters.

EC2, the oldest of the five services used, and AWS Backup, the newest, handle idempotence well. All five services handle idempotence differently! Inside AWS, the left hand did not know what the right hand was doing.

## Why Do We Care about AWS Inconsitencies?

Inconsistency is **expensive for AWS customers**.
I do not know whether AWS spent more by having each service team reinvent the wheel, or whether it earned more by bringing each service to market faster.
I do know that every customer using multiple AWS services has to discover their inconsistencies (sometimes by trial-and-error, because not every detail is or can be documented), write extra code to work around them, and then fix bugs that result from the extra complexity.

## How Can We Avoid Repeating this Pattern Ourselves?

As frustrating as inconsistency is to me as a software engineer, it is even more frustrating to me as an MBA. I offer this three-part solution for our own organizations: 

### A. Document Standards Centrally and Succinctly

I am not talking about a Confluence page tree of stale documents, requests for comment (RFCs), and so on. Nobody will read that. Anyone who does look will get discouraged because one author after another is inevitably "A former colleague".

The goal is **to foster awareness, not to cover every detail**. Fit the list of standards to one or two pages. Group the entries by topic. Limit each entry to one sentence. It is sufficient to write, "Dates and times: For code added after April, 2025, use ISO 8601 basic (no punctuation) format." If necessary, link to a detailed document or, better yet, to real code.

For the purposes of this article, you could write:

* If you prefer the EC2 approach:
  * "Idempotence: Repeated requests succeed, and the response mentions the initial state."

* For the Aurora approach:
  * "Idempotence: Repeated requests fail, and the error message mentions the initial state."
  * "Errors: Error codes match exception names."

* For the AWS Backup approach:
  * "Idempotence: Repeated requests succeed if the optional `IdempotencyToken` parameter, a string of 1 to _X_ arbitrary Unicode characters, matches."

### B. Review Past Practices when Designing and Coding

A review of past practices belongs in every important design, and also in the description of an important pull request (proposed code).

Add this to the "definition of done". Allocate time in proportion to the cost of changing the system later. For example, spend extra time looking for past practices when you are designing or implementing a public application programming interface (API), like AWS. Once customers and their code depend on your API, change becomes difficult, or practically impossible.

Do not let searching for past practices devolve to a perfunctory step, a mere checkbox in a template. If you found no past practice, mention where you looked and what you looked for. Documenting the absence of information is as important as documenting the information.

### C. Maintain the Standards

Even a short, succint list of standards can quickly fall out-of-date. Assign two members of each team to be responsible. Give them time, and recurring tickets, to edit their team's entries every quarter.  This is real work, worthy of recognition. It is some of the most valuable work in the organization, because of the time and money it saves. The savings may accrue to the organization itself, or to its customers.

Obviously, any person sepherding an important design or pull request should add to the standards list when setting a precdent, and update it when improving on a past practice. This is worthy of recognition, too. It is not, however, a substitute for informed, periodic editing of the list of standards.

We might not always recognize the consequences of our design and implementation decisions. That is okay. The goal is to **gradually build up habits of awareness, responsibility, and reciprocity**. My effort saves other people time, and their effort saves me time. As an organization, we try hard to prevent repetitive work. If it happens, our disciplined practices prevent different results, and especially worse results.
