---
layout: post
title:  "The Identity Dark Forest"
date:   2024-03-15 14:29:36 -0400
categories: itdr ai draft
---
Identity and Access management sounds pretty boring. It’s a somewhat crowded space with the major cloud providers (love or hate ‘em) selling their flavor of product with relatively similar implementation details.

There is an alphabet soup of “standard” or “best” practices that you may or may not follow, zero-trust, least-privileges, how’s your identity access governance? What about your identity security posture?

Additionally there is a growing population of tools related specifically to improving your identity and access security practices (or hygiene) that, in essence, are ensuring the thing your cloud provider gives you to protect access to the resources you rent from said cloud provider is configured and used correctly. Now, this sounds a bit ridiculous - shouldn’t the cloud provider help and guide you in securing the resources you build your business on? They definitely should, but they don’t.

Well, that’s not fair. They do an okay job of it; and the reality is, supporting a diverse set of customers and their needs requires IAM providers to support a degree of flexibility and be somewhat unopinionated about how you use their products. However, this results in customers broadly misconfiguring IAM policies as there are little to no guard-rails in place to protect from potential foot-guns. Arguably, if you know what you’re doing, you should be able to disable these guard-rails and go full IAM-cowboy. However, most companies would likely benefit from the added friction of a big red message preventing you from creating a new access policy that allows the `s3:*` action on all S3 buckets in your organization account.

Moreover, large companies with multiple teams working on different products, likely have a sufficient degree of complex IAM configurations, policies, and roles. This complexity only increases as these companies grow in headcount, use a broader range of cloud resources, use multiple cloud providers, introduce SaaS applications, and so on. Spotting these misconfigurations, which inevitably lead to attack paths, becomes a challenge - especially if your IAM provider does not offer native tools to automatically detect and remediate these costly mistakes. Even if they do, spotting non-linear attack paths that originate from third-party SaaS applications is increasingly difficult and most security teams rely on manually auditing access, which as you can imagine, is operationally expensive.

At Pensar we call this state of affairs, where identity and access data are scattered across heterogenous sources with limited visibility, the Identity Dark Forest.

## **The Dark Forest**

> “Where are we going?”
> 
> “To the darkest place.”

Identity and Access make up the widest attack vector available to a threat actor. It is the lowest hanging fruit and logically, even the most sophisticated APT (Advanced Persistent Threat) groups leverage identity-based attack chains to infiltrate infrastructure and establish a beachhead. This makes sense; it is expensive (both monetarily and temporally) to develop “traditional” zero-day exploits (perhaps less so with advanced AI capabilities, but more on that later) versus taking advantage of blind-spots in a company’s identity security perimeter and most importantly their employees - the leakiest link in your defense chain.

Threats in this vector can be hard to spot as they span both linear and non-linear attack paths. By this we mean, directly exploiting misconfigurations/bad IAM hygiene (linear) and compromising third-party applications, such as Github actions, to take advantage of access granted to these applications (non-linear).

Additionally, if an attacker is able to access, for instance, an organization’s AWS account, it can be difficult to detect this access in real-time using existing tools such as the AWS Access Analyzer as Access Analyzer only parses resource-based policy changes within 30-minutes of an update, triggered by CloudTrail logs, and during its daily scans. If an attacker is nimble enough they can move in and out of your infrastructure before Access Analyzer alerts you 30-minutes later. By then, as we explore in sections below, the damage can be critical.

> [!info] 
> Access Analyzer can be used to detect backdoors in your AWS infrastructure if used correctly (for instance by using the `aws:PrincipalOrgID` or `aws:PrincipalOrgPaths` keys to filter for access granted to external IAM principals).

 We believe an opinionated approach to identity and access security is essential to promoting widespread IAM hygiene. There are best practices out there however, the reality is most application developers do not want to bother with manually implementing said practices. They just want to set-it-and-forget-it when it comes to access controls (and this even extends to authentication) - which consequentially leads to best practices being ignored in most cases. If IAM providers took an opinionated approach to access control, (e.g. telling their users "hey maybe don't provide wildcard access to your dynamo tables to that one IAM role") aggregate identity security posture would be in a much better state than it is today.

Maybe this sounds like an easy fix, and perhaps you are in disbelief that this is even an issue. Fair enough. We thought so to.
[99% of cloud identities are too permissive](https://unit42.paloaltonetworks.com/iam-cloud-threat-research/?utm_source=2024-ir-report-Unit42-global&utm_medium=website)
[65% of detected cloud security incidents are caused by misconfigurations](https://www.paloaltonetworks.com/resources/research/unit42-cloud-with-a-chance-of-entropy)
(The Unit 42 group from Palo Alto Networks has a some great research on the topic)

<!-- [TODO: find a better source for the second link above] -->

This is why we take an opinionated approach to identity and access security here at Pensar. Our models are trained to detect a wide range of threats, however we suggest very pointed remediation actions. You shouldn't aim for the principle of least privilege, you should just implement it.

### **Adversaries and their arsenal**

> If I destroy you...
> what business is it of yours?

Below we outline a few of the dangers that lurk in the shadows of the identity dark forest. Some of these are opensource tools that can be used -today- to infect a victim’s cloud infrastructure, others are attack paths that have been exploited in the past. Maybe these are still used today, maybe they are not - but this is a small peek into the depths of the dark forest - once an attack path is identified, it’s very existence will draw forth the apex predators.


**AWS Endgame**

`endgame smash --service all`

[Endgame](https://github.com/DavidDikker/endgame) is an opensource command-line tool that exploits permission misconfigurations in AWS resources and/or IAM policies to grant rogue users and principals access to AWS account resources while creating backdoors to several of the most widely used AWS services.

The only prerequisite to using Endgame is an AWS API credential key - which could be obtained through various nefarious means including: advanced social engineering attacks, exploiting compromised Github accounts, scraping config files or credentials that have been mistakenly committed to a repository (you would be surprised how often this happens), and so on.

From there Endgame can run in either `smash` or `expose` mode.

`expose` mode surgically updates the resource-policy of a single AWS resource to include a backdoor for a principal owned by the attacker - or if the attacker is especially bold, exposes the resource publicly.

`smash` is significantly less subtle - targeting a single service or all supported AWS services, enumerating each active resource in the specified account region, and updating policies to grant access to a rogue principal. The attacker, depending on the degree of misconfiguration of the victim, may now insert backdoors that are not controlled by the victim’s IAM policies.

![[endgame.gif]]
Source: https://github.com/DavidDikker/endgame


Endgame was written three years ago, as of writing this. At the time Access Analyzer did not support auditing 11 of the 18 services targeted by Endgame.

As of today (March 2024), Access Analyzer still does not support 6 of the 18 services targeted by Endgame.


**Chameleon Attacks w/ AI**

CodeGen LLMs (i.e. language models trained on code) are being used in novel ways, one of which is UI generation (a la tools like [v0](https://v0.dev/) by Vercel). Pass in a prompt, and the model will output code for various frontend components conditioned on this prompt.

While this is a cute use case and great for our webdev friends, it is also possible for threat actors to use these tools to scale and perform chameleon attacks wherein an attacker deploys a malicious duplicate of a target site - typically an SSO or standalone sign-in page - that looks and smells like the real thing. These chameleon sites are used to steal credentials and can even mimic MFA authorization flows with high fidelity.

A recent example of such an attack is CryptoChameleon - a phishing kit used to target FCC employees by mimicking the legitimate FCC Okta SSO page. This phishing kit even requires the victim to pass captcha checks to both hinder automated threat analysis web-crawlers and lull the victim into sense of security.

The attacker can then tailor the malicious sign-in experience to match the authentication flow of the victim organization - manually adjusting input fields as needed as they use the victim's provided credentials to log into the legitimate site in real-time (learn more about this attack [here](https://www.lookout.com/threat-intelligence/article/cryptochameleon-fcc-phishing-kit?ref=news.risky.biz)).

This attack requires a malicious operator to react in real-time to user inputs. However, with function calling and agentic LLMs (learn more [here](https://platform.openai.com/docs/guides/function-calling))  it would not be difficult to replace this human operator with silicon.


**Advanced Social Engineering at Scale**

There are plenty of marketing blogs and posts that are hand waiving the impending risks of using LLMs for social engineering and spear-phishing attacks, not many highlight any practical experiments conducted in the wild against live targets. Luckily, researchers at UT San Antonio have recently published a study to the arXiv in January with data on a live red-team lateral spear-phishing campaign conducted against university staff using major LLM APIs to generate phishing emails.

Take a look at the results of the study [here](https://arxiv.org/html/2401.09727v1#S4.SS2.SSS3).

While the click through and data exfiltration rates are telling on their own, take a closer look at how the topic of said phishing emails relates to the "data entered" rate.


| Month     | Email Recipients | Emails Opened Link | Clicked Data   | **Entered**     | **Category of Phishing Emails**                                         |
| --------- | ---------------- | ------------------ | -------------- | ----------- | ----------------------------------------------------------------------- |
| Nov. 2023 | 8,995            | 4,461 (49.59%)     | 1,501 (16.69%) | **613 (6.81%)** | **Event Coordination Scam: Holiday Party Planning, New Dining Options** |
| Oct. 2023 | 8,856            | 5,303 (59.88%)     | 2,352 (26.56%) | **646 (7.29%)** | **Conference Pictures Request, Internal HR: Urgent Request**            |
Source: [*Large Language Model Lateral Spear Phishing: A Comparative Study in Large-Scale Organizational Settings*](https://arxiv.org/html/2401.09727v1)

The two phishing emails with the highest rate of "data entry" (meaning victims provided sensitive data) pertained to topics unrelated to daily operations, credentials (i.e. malicious "please reset your password" notifications), and what would commonly be associated with sensitive data (one phishing email included in the study pertained to W2 verification).

There are other factors at play here (see section 5 of the above paper) however, these results are still quite telling.

Would your employees spot these emails as phishing? Would you think twice before responding to a mundane request from HR about conference pictures or a new dining option? These topics may seem benign - what could an attacker possibly get from a phishing email about meatloaf versus pizza in the office cafeteria? You would be surprised. Especially if the linked malicious survey required users to "sign-in" to a spoofed internal domain to record their preferences. After all, some employees may feel more strongly about preventing meatloaf's infiltration into their organization's dining options than preventing a malicious attacker from infiltrating their network.

What is even more interesting is how easy it is to scale such attacks horizontally across an entire organization. In the above paper, researchers targeted an entire 9,000 person organization with relative ease.

**The use of highly capable LLMs unlocks attractive unit economics for threat actors.**
<!-- [perhaps an atlas map here of threat-actor llm calls, etc.] -->

## **The Identity and Access Defensive Perimeter**

> There are no permanent enemies or comrades, 
> only permanent duty.

Hopefully it is clear now that relying on your employees alone to defend the access perimeter is not sufficient to deter and fend off attackers. Employees are, unfortunately, your biggest liability in this sense. No amount of "cyber-awareness" training will turn your entire workforce into hardened anti-cybercrime vanguards. It is arguably irresponsible to rely so heavily on employees as a line of defense in the first place, as their priority should be to, well, their jobs. In order for that 10x engineer or that killer salesperson to continue to perform, the employer should create a sufficiently frictionless environment to do so. Having to act as a human threat detection layer increases operational friction.

It should also be clear that even the most seemingly benign IAM misconfiguration error can lead to critical damage dealt to your organization's cloud infrastructure.

As such, it is imperative that the modern enterprise invest resources in hardening their identity and access perimeter. It is the [Holtzman shield](https://en.wikipedia.org/wiki/List_of_technology_in_the_Dune_universe#Holtzman_effect) that limits the impact radius of an attacker.

Identity and access defense begins with visibility. You cannot protect what you cannot see.

IAM data (policy statements, privilege definitions, access grants) reside across heterogeneous sources, in companies with sufficient size and infrastructural complexity. These sources include cloud providers/environments, SaaS applications, internal applications or databases, on-prem systems, etc. Without a common fabric to connect all of these sources and provide a single pane of glass to view this data, security teams must manually parse and audit these sources. At a large organization - this is not only impractical, it is unsafe. Attackers will outpace your audit processes, your vulnerabilities will not be spotted in real-time.

Having a unified view of access across these sources (Opus represents this view as access trees for each employee, team, etc.) removes the need for manual parsing of this data and gives security teams a holistic visualization of access and identities across their organization.

A common visualization layer alone is not enough. Detecting erroneous or anomalous access (this comes in many flavors - configuration errors, orphan accounts/access, overly permissive privileges) is still a manual burden. Access policies and privileges are commonly assigned "by hand" and, as discussed earlier, the onus is generally on the customer to have good IAM-hygiene and not enforced by the provider or application.

This is where machine learning (pardon, AI) is especially effective. For instance, at Pensar we train and use ML models to detect gaps in the identity and access defensive perimeter in real-time. These models are able to flag, for example, privilege misconfiguration errors across hundreds or even thousands of access policies and roles (the same kind that Endgame exploits) and suggest changes to these policies that actually adhere to proper IAM hygiene.

We may now augment the security analyst with a silicon companion. One that is constantly and deeply scanning for defensive vulnerabilities, and not only alerting the analyst in real-time but proactively prompting them to action.

<!-- [visualization of Opus/models detecting IAM errors, suggesting fixes - a la git diff ui] -->
Automation and machine learning have been deployed to modern endpoint and network protection solutions. It is time that identity and access security likewise evolves.

If any of the above got you excited or scared, check out our company PensarAI. We are building the Holtzman shield for identity and access security.

Kerem Proulx
<!-- [signature here] -->