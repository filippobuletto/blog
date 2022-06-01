---
title: "Application Security: why prevention is better than cure"
excerpt: "My thoughts on application security, why it is important not to underestimate it."
permalink: /application-sec/
header:
  overlay_image: /assets/images/internet-security.jpg
  overlay_filter: rgba(0, 0, 0, 0.5)
categories:
  - Programming
tags:
  - security
  - development
toc: true
toc_sticky: true
toc_icon: "shield-check"
---

# Introduction

**Disclaimer:** This article first appeared in Italian on [futurodigitale.infocert.it](https://linkanonymous.com/?https://futurodigitale.infocert.it/cybersecurity/application-security-perche-prevenire-e-meglio-che-curare/)
{: .notice--info}

Attacks directed at application security are the main source of breaches that the IT industry sustains in a world that, to date, is highly interconnected and dependent on technological systems.[^1]

Spending time and money up front on application security can significantly reduce costs and risks after the fact, but it does not always turn out to be the main strategy deployed by IT companies.

Why is this? The trade-off lies in the fact that time and money spent on preventive security is time and money that cannot be spent on accelerating the pace of software delivery. Spending is often reduced as long as security and risks are perceived to be at a level acceptable to customers, partners, and management.

The key then is to determine the right balance between proactive control, reactive agility, tolerance for risk, selection of the right tools and practices to make applications secure without slowing down development. We will then see how the perception of risk and benefits related to this good application security practice plays a key role in its application.

In this article, we will learn about the application of proactive controls and interventions through an analogy that can help even the less experienced in the field understand why this is a very important as well as underrated tool.
Information system as a living organism

What is an "organism?" Let's look at the definition from the Treccani dictionary (translated from Italian):

> In biology, a living being, animal or plant, endowed with its own specific form, cellular structure, and consisting of a set of organs interdependent and in such functional relationship as to render it capable of living autonomously, that is, of conserving and possibly replenishing its own form, and of reproducing itself
>
> Organized system, consisting of various interrelated and interdependent parts

It is immediately apparent how the definition is very close to that of a computer system: an organized and structured system, connected and interdependent parts, and the ability to preserve its own form/integrity.

The analogy does not end at the definition; in fact, it turns out to be applicable in the field of computer security as well, specifically with regard to the attack on the computer system, which consists in a nutshell in an external "malicious" act aimed at altering the system and removing high-value parts (the data!).

Thus we see how the system-organism must be provided with what we might call an "incentive" for the immune system, a natural shield always ready to intervene (both passively and actively) in case of external aggression.

## Security in times of pandemic

Never before have we been exposed to concepts such as that of a virus, or malevolent agent, capable of doing serious harm to our bodies, understanding what a key role disease control and prevention plays in modern society.

The attack of a pathogen is often as unexpected as it is violent and dangerous, and when it manifests itself in these ways, it can cause serious damage to the point that any measures to contain the problem are often ineffective or, worse, late and useless.

That is why the most powerful weapon that medical science has produced to counter the action of these malevolent pathogens is a proactive system: the vaccine[^2]!

Here we come to the main analogy: proactively intervening in cybersecurity before and during the application development cycle is totally comparable to using a vaccine to defeat the external malevolent agent that wants to attack our system-organism!

Let's look at some parallels:

- The developing computer system is healthy, it is not (yet) sick, as the vaccine is inoculated to healthy organisms even the controls are developed without an actual attack taking place.
- Controls and interventions are part of the system, not external, just as the vaccine boosts the body's immune system by protecting it without the intervention of other medicines.
- Proactive controls and interventions come at a cost; even vaccine administration is expensive (in Italy indirectly, through the National Health Service).
- The use of proactive controls is effective in the long run and promotes the use of "Best Practices," as vaccines are not limited to improving clinical situations but are also able to promote economic growth in countries that use them[^3].

Although they are both shared and obviously useful concepts, there is widespread hesitation about their use, and one of the main reasons is: the perception of risk and benefit.

## Prevention seems counterintuitive but turns out to be effective

What is risk perception?

> Risk perception is a cognitive process involved in a variety of everyday activities that guides people's behaviors when faced with decisions involving potential risks. Risk perception involves several dimensions such as, for example, both immediate and future consequences and their implications on both a rational and objective level and on an emotional and subjective level[^4].

We thus see that the human predisposition is to handle risk instinctively and emotionally, the common tendency being to perceive as negative something we know little about (or of unknown nature) and as positive something with which we are familiar (but which is objectively worse).

Although the perception is often one of skepticism, both toward vaccines[^5], and proactive controls[^6], the evidence is that these are extremely effective measures for keeping our systems-organisms healthy and safe from external attack.

That is why, in order to move from an intuitive and negative assessment to an objective and positive one, it is necessary to delve deeper into these proactive controls and interventions, finding out how and when to implement them and what benefits they will bring to systems by making them "secure by design."

So what is an effective way to equip yourself with these controls and ensure that they are implemented throughout the software development cycle? Evolve your Software Development Life Cycle (SDLC) into a "Secure Software Development Life Cycle" (SSDLC), a topic we will explore in more detail in the next article in this series.

[^1]: [Data Breach Investigations Report 2021](https://www.verizon.com/business/resources/reports/dbir/2021/masters-guide/)

[^2]: [Il valore scientifico e sociale della vaccinazione](https://www.vaccinarsi.org/scienza-conoscenza/vantaggi-rischi-vaccinazioni/il-valore-scientifico-e-sociale-della-vaccinazione)

[^3]: Bloom DE. The value of vaccination. Adv Exp Med Biol. 2011;697:1-8.

[^4]: [Percezione del rischio](https://www.dpss.unipd.it/JDMLab/aree-di-ricerca/percezione-del-rischio)

[^5]: [Centers for Disease Control and Prevention. Impact of Vaccines in the 20th & 21st Centuries. The Pink Book: Course Textbook – 13th Edition 2015.](http://www.cdc.gov/vaccines/pubs/pinkbook/index.html)

[^6]: [OWASP Proactive Controls](https://owasp.org/www-project-proactive-controls/)