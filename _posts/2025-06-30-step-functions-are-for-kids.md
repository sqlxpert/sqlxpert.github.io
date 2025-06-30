---
layout: post
title: "Step Functions Are for Kids"
author:
  - Paul Marcelin
# image: /assets/images/UNKNOWN.jpg
# image_alt_text: UNKNOWN
---

_Amazon Web Services Step Functions are suitable for "real" work and they would also make a fine introduction to programming. Maybe job security is the reason low-infrastructure, low-code, visual paradigms are so threatening to macho software engineers..._

---

A user of my open-source software brought up a need that was new to me. I searched GitHub, Medium, and Amazon's re:Post knowledge base. Existing solutions would have worked most of the time, but they suffered from a latent timing bug that would eventually have caused a failure &mdash; a failure that could cost money. I wrote a reliable solution in 333 lines of Python, packaged it, and published it. It earned a mention in the [Last Week in AWS newsletter](https://www.lastweekinaws.com/newsletter/) (issue 427, not yet archived), garnered a few stars on GitHub, and that was that.

Then, I did something heretical: I tried re-writing it with "zero code".

## Resist Change, at Any Cost

The small startups where I've worked encouraged new ways of doing things, but most of the large companies would not have allowed this, at least not in the past. I met crushing resistance every time I brought up modern, serverless, low-infrastructure, low-code architecture.

I started programming in [Texas Instruments 99/4A](https://en.wikipedia.org/wiki/TI-99/4A) BASIC before I was 10, taught myself 65C02 assembly language on the Apple II while in middle school, programmed Cray supercomputers in high school, and earned a degree in Logic and Computation from Carnegie Mellon, so if I suggest a visual programming paradigm like Step Functions, it's not because I'm too stupid to do it the normal way. That was the first assumption certain big-company colleagues made. (So what if someone actually does start with visual programming? I am thankful for my experience and education, but there are alternative ways to learn.)

Next came the fear that "little Lambda functions" would proliferate. (AWS Lambda is a cloud computing service for running short programs. I capitalize Lambda because it's not the same as the math concept. Ditto for Step Functions.) Never mind the proliferation of scripts in big companies! Scripts are terrible for reliability and maintainability. Whereas automated checking is the norm for program code, I rarely saw scripts that would have passed [`shellcheck`](https://github.com/koalaman/shellcheck#shellcheck---a-shell-script-static-analysis-tool)&nbsp;. How do you test your script? Is there structured and central logging, or does it just write plain text to standard-error? Where's the documentation?

Worse yet, server images or container definitions proliferate along with scripts, because every script needs an environment in which to run. Nobody wants to go back and update the operating system and the software packages that were frozen in time when the script was written. Scanning tools that automatically spit out security update tickets make the maintenance burden hard to ignore. Yet, for many software engineers and managers, scripting epitomizes "real", grown-up work. It's still a major part of most interviews. Yes, we need to know how, but managers should be wary of software engineers whose first impulse is to write another shell script.

The last reason to block low-infrastructure, low-code architectures was pathetic, in the true sense of the word. Read the next sentence in a pouting voice. "But, we don't have a deployment pipeline for Lambda." Translation? 'The conglomeration of shell scripts and Jenkins plug-ins I made so our manager could boast about releasing multiple times a day and I could get promoted from senior to staff level, can't handle that.'

I spent a Saturday building a suitable continuous integration/continuous deployment pipeline in my personal AWS account, to gauge how difficult it would be. On Monday I reported, happily, that I'd been able to set up an experimental pipeline myself. My colleague's facial expression shifted and he became very defensive. "I've done that, too." I had hit a nerve.

> In the classic 1967 film _The Graduate_, just as Mrs. Robinson (Anne Bancroft) is about to complete her seduction of Benjamin (Dustin Hoffman), the vaunted college graduate gets cold feet.
>
> "Look, maybe we could do something else together. Mrs. Robinson, would you like to go to a movie?"
>
> Exasperated, she thinks for a moment and settles on one thing that will provoke him: questioning his masculinity.
>
> &mdash; Quote from _The Graduate_, 1967. Screenplay by Calder Willingham and Buck Henry, based on the novella by Charles Webb. 50th anniversary Blu-ray edition, StudioCanal, 2017, 0:37:53.

I was excited that the recipe in AWS's documentation worked, and that AWS's own CI/CD tooling could do the job. Nothing extra to install, no [Jenkins](https://www.reddit.com/r/devops/comments/12fhdkm/what_are_the_real_cons_of_using_jenkins/)! I, the dumb one who liked "little Lambda functions", certainly didn't intend to bruise my colleague's ego, but if that had been my intention, what better way? 'You mean, _you_ haven't tried this yet?'

I began writing software professionally in 1993. I still haven't figured out what macho software engineers get out of their behavior. Does it feel good to ignore other people's ideas? To block change? To write more code than necessary, make it more complex than necessary, and use more infrastructure resources than necessary? Maybe it's a job security ploy, a way of making oneself essential.

## What's a Step Function?

An AWS Step Function resembles a board game. The boxes are "states". You say what to do in each box, connect the boxes with arrows, and specify conditions for following the arrows. I'll use [Monopoly](https://en.wikipedia.org/wiki/Monopoly_(game)#Board) as an example. The "Community Chest" box is a "Task" state; your task is to pick a "Community Chest" card. Every card represents a "Choice" rule. If you get the "Advance to 'Go' and collect $200" card, you move to the "Go" state, which is another "Task" state, your new task being to collect the money.

You can define a Step Function visually, by dragging-and-dropping states and filling in the details, or you can write the JavaScript Object Notation (JSON) definition yourself.

[<img src="/assets/images/aws-step-function-visual-editor-20250630.png" alt="New flow states including Choice, Parallel, Map and Pass can be dragged from the palette on the left to the Step Function diagram on the right. The diagram consists of states connected by arrows." height="144" />](/assets/images/aws-step-function-visual-editor-20250630.png?raw=true "Automatically-generated Step Function state machine execution diagram")

## Is Zero Code Possible?

The best of the existing solutions to my user's problem was written by an AWS employee. It has two latent timing bugs, but in all other respects, it is a thorough and thoughtful solution.

What struck me is that it is far from zero-code. It's a Step Function, yes, but three of the "Task" states run Lambda functions. Lambda is a big step in the right direction, but you still have to maintain each function's Python code. The Lambda service charges you for memory and total computing time, not major factors in this case, but important to be aware of.

My goal was to solve the problem without Lambda functions, that is, without the need to maintain Python code.

Two features make this possible. You can [run most AWS commands inside a Step Function](https://docs.aws.amazon.com/step-functions/latest/dg/supported-services-awssdk.html) and you can process the results using a query language called [JSONata](https://jsonata.org/). ([JSONpath](https://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-paths.html#amazon-states-language-reference-paths), an older option, would have worked too.)

In computer science, there is a distinction between imperative and declarative programming. Familiar computer programs are imperative: they have branches (if-then statements) and loops, and each step can affect the outside world (a so-called "side effect"). Think of printing a page one line at a time. Declarative programs specify the goal without telling exactly how to achieve it. Only the last step should affect the outside world. Think of composing the lines one by one and printing the whole page when it's ready.

Declarative programming saves effort. It is the zero-code ideal.

Some say that Step Functions are declarative. That's an exaggeration. "Choice" rules allow the same branching and looping as in ordinary, imperative programs, and "Task" states can affect external systems. Though Step Functions fall short of fully declarative programming, you get diagrams that show which states were traversed, and you can easily check what was done by each state. (To accomplish these things for traditional code, you'd need extra software, in the form of a high-quality graphical debugger.)

Within a state, you can use JSONata imperatively, by writing blocks with multiple statements, or declaratively, by moving data through successive transformations. My goal became to write short JSONata expressions consisting of [transforms](https://docs.jsonata.org/other-operators#-------transform) &mdash; if not zero code, then low code.

## Faster Programming and Testing

Replacing more than 300 lines of Python with fewer than 200 lines of JSON/JSONata is a miracle. Imagine the impact on a larger scale!

I wrote part of the Step Function definition manually but used the visual editor to insert and delete states. At the end of the process, I edited the JSON code for readability.

The automatically-generated diagram grew complex. Some of the complexity was necessary: more states and more arrows, to handle errors. Some of it was unnecessary: arrows that could be drawn without crossing, if only the drawing module were more intelligent or could produce its output in an editable form, for humans to refine. Yet, even the complex diagram became eminently readable when I tested the Step Function. States traversed were highlighted in green (success) or yellow (error). I could tell at a glance, without zooming in, how a test had turned out.

[<img src="/assets/images/aws-step-function-debug-flow-20250630.png" alt="After a 'Wait' state, a 'Choice' state chooses between cluster or instance, if the event has not expired. Because this event is for a database instance, a 'Task' state that calls 'Stop Database Instance' is entered. The other states described are green, whereas this one is yellow. From an arrow labeled 'Catch #1', execution continues with a 'Task' state that calls 'Describe Database Instances' followed by a 'Choice' state that chooses between different database status values. Because the database is in the desired state, the 'Succeed' state is entered." height="144" />](/assets/images/aws-step-function-debug-flow-20250630.png?raw=true "Automatically-generated Step Function state machine execution diagram")

## Lower Total Cost

With Step Functions, you pay by the arrow. Traversing 10,000 arrows costs 25¢, regardless of the total time spent, and regardless of the amount of memory used inside each state. You are allowed to push up to 256 kibibytes of data (about 145 pages of text) between states. You pay normal charges for logging, and for calling other AWS services from "Task" states. ([Pricing](https://aws.amazon.com/step-functions/pricing/#AWS_Step_Functions_Standard_Workflows_pricing_details) and [limits](https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html#service-limits-task-executions) can change. Pricing varies by region.)

There is no server to pay for, and no operating system to update.

A cost overlooked by macho software engineers and the managers who abet them is the labor cost incurred when colleagues and successors have to operate and maintain overly complex software. Step Functions are potentially easier to understand than traditional programs. The Step Function diagram is weak as an explanatory tool but becomes useful once you make a test run and see the states traversed highlighted in green.

In the end, though, the developer of a Step Function should spend time on documentation and a custom diagram. Step Functions require less documentation but they are not fully self-documenting.

## Less Control Over Retries and Logging

Two things I dislike about Step Functions are limited control over error handling and logging.

If your Lambda function or traditional Python program runs AWS commands, you need only [set one parameter to get automatic retries](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/retries.html#standard-retry-mode) for 18 specific and 4 general errors. Step Functions support retries, but achieving the same diligent retry logic would be impractical. First, you'd have to use trial-and-error to find the exact names of the 22 errors, because error names are inconsistent and there is no single document. Then, you'd have to repeat the list of 22 errors in every "Task" state!

Fortunately, I was solving a watch-and-wait problem. In this particular Step Function, no single AWS command execution is essential, because the request will be repeated minutes later.

As for logging, logs should be kept clear so that critical problems stand out, if the Three Mile Island nuclear reactor accident is any guide.

> "The computer printer registering alarms was running more than 2½ hours behind the events and at one point jammed, thereby losing valuable information."
> &mdash; [_Report of the President's Commission on the Accident at Three Miles Island_, October, 1979, p. 30](https://archive.org/details/three-mile-island-report/page/30/mode/1up)

The Step Function service logs all errors with the same priority, instead of giving me the option to log expected errors (such as stopping a database that's already stopped, the yellow state in the test run shown above) at a lower priority so you can ignore them. Because the log levels jump from ERROR to ALL, there's also no automatic way for me to log a successful run without throwing in irrelevant data.

## Portability Is Probably a Red Herring

I dealt with objections from colleagues earlier. Managers might object that Step Functions and other low-code, low-infrastructure programming paradigms are not portable. Neither is the shell script, nor the traditional Python program! When you switch cloud providers, utility software has to be rewritten. The commands have different names and different parameters. Perhaps the utility isn't needed because the service works differently or, in the case of a cost-saving utility, because the pricing basis is completely different.

(Investors, executives and the finance department should question claims about portable application software, too. Is writing software that runs on three different cloud providers your core business, or a technical distraction? What are the opportunity costs? What isn't getting done while your staff-level software engineers are configuring and upgrading [Kubernetes](https://www.socallinuxexpo.org/scale/21x/presentations/terrible-ideas-kubernetes)? Which of your main cloud provider's native performance and cost optimizations and security idioms are you sacrificing when you design to the lowest common denominator?)

## Democratizing Software Development

You can compare my two solutions to the problem of keeping an AWS database stopped at [github.com/sqlxpert/**step**-stay-stopped-aws-rds-aurora](http://github.com/sqlxpert/step-stay-stopped-aws-rds-aurora) and read about the strengths and weaknesses of other solutions (including AI-generated code) at [github.com/sqlxpert/stay-stopped-aws-rds-aurora#perspective](http://github.com/sqlxpert/stay-stopped-aws-rds-aurora#perspective)&nbsp;.

Earlier, I rattled off a list of personal accolades to counter the assumption that only an inexperienced programmer would bother with Step Functions. Step Functions are suitable for "real" work and they would also make a fine introduction to programming. Maybe job security is why low-infrastructure, low-code, possibly visual paradigms are so threatening to macho software engineers.

Step Functions, etc. pose a more direct competitive threat than artificial intelligence / large language model (LLM) code generation. A novice writing a Step Function has to practice reducing a problem to discrete steps, and think about how those steps fit together. The person is on the way to learning good programming practices and some computer science. A novice reviewing AI-generated code can't discern semantic errors in that code, and isn't necessarily learning.

Me, I'm in favor of democratizing the software engineering profession. If there are more people who can do what I do, I feel comfortable leaving today's problem in their hands. I enjoy supporting their learning and growth, as I turn toward tomorrow's problem and learn about that...

> Have you had great results with a technology, a method, or an idea that other people made fun of? Tell me about it by commenting on the [LinkedIn version of this article](https://www.linkedin.com/pulse/step-functions-are-for-kids-paul-marcelin-UNKNOWN)!

