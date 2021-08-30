---
title: "SOX Policies in Guardium Insights"
date: 2021-08-01T18:15:07-04:00
draft: false
tags: ["sox", "privileged_user", "data_integrity", "guardium_insights", "policies"]
---
*Author: John Haldeman*

In our [previous post](/posts/sox-auditing) we talked about SOX
reporting. In this post we'll discuss what a SOX policy might look 
like for similar requirements.
<!--more-->
In that previous post we had [a section about SOX requirements ](/posts/sox-auditing#requirements). 
For this post, let's build a policy to meet the same requirements. That is
to monitor changes performed by privileged users defined as follows:

| Audit Definition      | Description |
| ----------- | ----------- |
| Privileged Users | Any execution involving a source program, user, or client IP (or hostname) not associated with an application or batch service |
| Changes   | DML and DDL        |
{{% caption %}} Definitions for our SOX example {{% /caption %}}

Guardium policies work with rules that are evaluated sequentially 
according to criteria. If the criteria are met, a specified action is
executed. For this requirement our criteria will be the 
definitions for privileged
users and the commands involving changes (DML and DDL). Our action 
will be to either log the SQL or ignore the session entirely. 
There is an assumption here that that the number 
of SQL should be manageable because the application and batch 
traffic is filtered out.

**Note:** This example contains an ignore list. That means we will
explicitly define what to ignore, and log everything else. This is
obviously a good security practice, but it does mean that the
administrators of the environment will have to keep on top of the
ignore lists. The good news is, I have maintained those lists and,
for even very large environments, it is not impossible or even
difficult to do with the right kinds of processes in place. We will 
explore what that looks like in a different post. For now, let's 
build the policy.

There are a lot of ways to model these same requirements in a
Guardium policy. Let's pick a common way to do this and use 
that as an example implementation. At the end of the post I'll 
talk about a couple of alternatives.

## Policy Outline: Ignore List with a Logging Rule Below It

In our implementation we'll have one rule that describes the 
traffic we do ***not*** wish to log. It blocks execution of the 
rule following it which does log the traffic we want. It looks 
as follows:

1. If Source Program is associated with an application or batch 
service AND the DB User is as well AND so is the Client IP is too, ignore
the traffic (alternatively log less using construct logging [^1]). Stop 
evaluation or rules here.
2. If the command is DML or DDL, log the data

Note that we only get to rule 2 if we pass rule 1. Rule 1 defines the
situations we wish to ignore

## Before We Begin: Groups

In this tutorial we are going to need some groups. The groups define
the nouns of the rule descriptions above. We're going to use the same
groups we created in our previous post of SOX reporting. You can find
the instructions for doing so in [the Groups section of that post](/posts/sox-auditing/#groups).

## Implementing the Policy in Guardium Insights
Now that we have an outline of what the policy should look like,
let's go step by step and create the policy in Guardium Insights.
Before we go into the details, do note that policies in Guardium
Insights only apply to the traffic captured directly inside Guardium
Insights. Currently that's only direct AWS Kinesis, Azure Event Hub,
and Universal Connector feeds. Watch this space though in case the 
available feeds increases. That said, the policies in Guardium Insights are very
well aligned with Guardium Data Protection, so what you see here you
could also use in a more traditional Guardium environment.

The first step is to access the policy builder. Click on the Menu and
then select Policies

{{<image src="/images/policies-nav.png" caption="Navigating to the Policy Editor: Menu --> Policies" >}}

Next, let's click on the "Create a policy" button.

{{<image src="/images/policy-create.png" caption="The 'Create a policy' button is to the right of the screen" >}}

Next under "Create a custom policy" name the policy something like
"SOX Policy" and click "Create" at the bottom of the screen.

{{<image src="/images/polict-create-name.png" caption="Name the policy" >}}

This is the policy editor. On the left you have what are known as
"Access rules". These are the rules based on SQL executed or 
connections made to the databases you are monitoring. On the right
are "Exception rules" which define actions on failed logins or SQL
errors.

{{<image src="/images/policy-builder-page.png" caption="The policy editor" >}}

Let's add our first rule - the one that defines what traffic we 
do not want to log that will prevent execution of our second rule.

{{<image src="/images/policy-add-access-rule.png" caption="Add an access rule" >}}

Our criteria for the rule defines what application and batch traffic looks
like. In our requirements, that is traffic that is from an application
server IP, a known application source program, and uses a known application
database user. To do that, add three criteria to the rule using the groups
we built for the reports (see the screenshot below). AND logic is applied inside Guardium policy rules.
That is, all criteria must be satisfied to trigger the rule action. If you
want to apply OR logic, you would need to create multiple rules.

{{<image src="/images/policy-add-conditions.png" caption="Our first policy rule with the three criteria added" >}}

Next, we define the action to take. We have no other requirements for our simple
example, so let's choose the rule action that will result in the least performance
impact in our environment - that is "Ignore Session". If you've used Guardium for
awhile, you'll notice a lack of "Ignore STAP Session" rule. That's because STAPs
can't report to Guardium Insights directly just yet.

{{<image src="/images/policy-session-action.png" caption="IGNORE SESSION action with 'Stop evaluation' (ie: if the criteria is met, no further rules will be evaluated)" >}}

Click the Add button to add the rule to the interface. You can expand the rule to
view its behavior at a glance once it is added.

{{<image src="/images/policy-added-rule-1.png" caption="Our first rule" >}}

Now, add another rule. This one just for DDL and DML statements with the action
"Log Full Details". 

{{<image src="/images/policy-dml-ddl-rule.png" caption="Log Full Details for DML and DDL" >}}

That's our policy! Two rules to fulfill a simple requirement. Make sure you click
the save button so that the policy is saved.

{{<image src="/images/policy-done-save.png" caption="Our two-rule policy. Be sure to click save!" >}}

You may be wondering what happens in situations that meet neither rule's criteria (for example: A DBA
running a SELECT would meet neither rule's criteria). People
who know about Guardium know that it depends on what kind of policy this is - a 
non-selective audit trail or a selective audit trail. You may have noticed that there
is no option to select what kind of policy this is in Guardium Insights. That's because all policies in
Guardium Insights are non-selective audit trails. These policies log 
constructs[^1] when no rule is satisfied. If you're translating a selective audit trail
from traditional Guardium to Guardium Insights, you'll have to change some of your
rule actions and add a new rules to simulate a selective audit trail[^2]:

The next step is to go back to the policy list and then install the policy we have created.
Once at the policy list, click the "Manage policies" button in grey.

{{<image src="/images/policy-after-create.png" caption="Click 'Manage policies'" >}}

We'll deactivate the currently installed default policy and activate our new one. If you 
have multiple policies activated, this is also where you can specify the order by dragging
the policies around.

{{<image src="/images/policy-install.png" caption="Activating the SOX policy and deactivating the default one." >}}

Now we just need to confirm policy installation.

{{<image src="/images/policy-install-confirm.png" caption="Click confirm to apply the policy change." >}}

That's it! We've created our policy that reflects our requirements and installed it. 
Before we end though, let's revisit our reporting with our policy in mind and also
talk about other ways you may have implemented these requirements.

## Revisiting Our Reports

I [mentioned briefly](/posts/sox-auditing/#next) in the post on SOX 
reporting that there would be some overlap in data between our three reports.
That isn't a compliance issue, but it is an efficiency issue. You may not want to
spend the time reviewing the same records twice or even three times. One way to
get around this is to build a more powerful query editor (which is hopefully on the way).
While we wait for that, you can use what you know about the policy to make the reporting
simpler. Our policy prevents the logging of SQL for the same DB Usernames, Source Programs, 
and Client IPs we filter out in the reporting. That makes those filters redundant, which
means we can remove them and consolidate the reports into a single report (you would keep
the filter to only show DML and DDL statements because you would log SELECTs with our
policy as well - just not at the Full SQL level).

Now, if you have a more complicated policy that logs a lot of different
things and only want to report on certain data logged by certain rules, a good trick is
to make use of the "Rule Name" field and filter for the specific rule there 
(note, in Guardium Data Protection (GDP), this field is called "Access Rule Description"). This allows
you to skip redoing all of the criteria that you have in your policy, inside your reports
as well. Note though this will only work with reports built on the Full SQL
Category  (policy violations would also work if you alert or LOG ONLY).


{{<image src="/images/filer-on-rule-name.png" caption="Rule name - Handy if you want to filter data to a specific policy rule." >}}

I have even occasionally had reports run a lot faster by filtering on this field 
instead of replicating criteria that exists in the rule, inside the report (well, 
in GDP at least and particularly when wildcards are involved).

Now let's talk about some other ways you could implement these requirements in the policy.
Below are two that come to mind:

## Alternative Policy Approaches: 
As promised at the beginning of the article, here are a couple of
alternative policy designs that will fulfill this requirement.

### Alternative 1: Split the ORs
Here we split each OR criterion into its own rule. This option 
would have a 3-rule policy that looks as follows:

1. If Source Program is NOT associated with an application or batch 
service, and the command is a DML or DDL statement, then log it
2. If DB User is NOT associated with an application or batch 
service, and the command is a DML or DDL statement, then log it
3. If Client IP is NOT associated with an application or batch 
server, and the command is a DML or DDL statement, then log it

This has the advantage of not having the ignore rule in the mix.
Sometimes in more complicated situations you may not want 
interactions where one rule blocks another
and in those instances, something like this makes more sense.
You'll notice that this alternative also looks a lot like how our
reports where built. There's something to be said about having
a certain amount of policy-report symmetry (for example - if you
log something and don't have a report on it and a process to review
the data, it's a question to ask yourself if you should be logging it).

### Alternative 2: Use Tuples

One could model these requirements using what's known as 
5 or 7-tuple criteria. These are concatenations of fields together
separated by a '+' delimiter. For example, a 5-tuple 
"Client IP/DB User/Source Program/Server IP/Service Name" type might have
a tuple like 10.10.9.100+JOHN+APP1+10.10.9.101+ORADB1. Some people
really like tuples because it gives them a lot of control. You
can model our exact requirements with judicious use of wildcards (%) in the
tuples (eg: to ignore all sessions with JOHN as the user, you could use the tuple
 %+JOHN+%+%+%). You could also specify full tuples without
wildcards for specific situations that you want to make exceptions for.

Using tuples, you could actually condense our entire requirement
into a single rule - which might be helpful if you have an already
complicated policy.

All that said, tuples are not yet supported in Guardium Insights, so
that decision is really more one for someone implementing things in GDP
right now. Also reading a list of tuples and discerning behavior can
be more difficult.

## A Note on Templates

Before we end, I just wanted to take a moment and talk about templates.
Guardium has always featured a large number of template definitions but,
in my experience, they are very seldom used.
It is interesting to think about "why". In writing these two articles I think
I have come up with a working theory. As we saw in the article on developing reports,
there are [many variations](http://localhost:1313/posts/sox-auditing/#requirements) 
on what SOX compliance means. I think the variation stems from the many different definitions
on the nouns in the requirements (the general themes are always about the same). 
Then you combine that variation with the many
different ways of implementing a requirement. None of them are incorrect,
but instead either more appropriate or less appropriate depending on the situation and the
environment. All of that together makes every Guardium deployment
somewhat unique, which has been my experience and the experience of 
many other practitioners that I have spoken to.

So, what's the use of templates at all? Well, if you are alone, stuck, or
just want to see how your approach compares to the template to help generate
new questions for the auditors, they can be helpful. Just because they
don't get used very often or directly, doesn't mean they are not helpful
in some situations.

## Next Steps

We're not quite done with SOX yet. We have a working policy and report set 
for SOX requirements in Guardium Insights. In the next post we'll see what
we can do to try help make review of the data a little easier for our compliance
team.

## Conclusion

In this post we looked at how to implement a policy for some example SOX
requirements. After that we discussed how reporting and policies can affect
each other and help each other, looked at alternatives to the policy's design,
and discussed a theory on why templates may not be used often with Guardium.






[^1]: Construct logging is a way of logging dramatically less 
data while avoiding ignoring the traffic entirely. It is 
appropriate in cases where you have a requirement to log application
or batch SQL (connections (log ins/outs) are always logged). More 
details on construct logging will be presented in a future post.

[^2]: To translate Non-Selective Audit Trails to a Selective Audit Trails and 
keep the same behavior make the following modifications: 1) Create 
a "SKIP LOGGING" rule at the end of policy with no criteria. 2) 
Change all the actions of rules that have "ALLOW" actions and make them 
"SKIP LOGGING" instead and 3) Change all rules with "AUDIT ONLY" actions 
to "ALLOW".



