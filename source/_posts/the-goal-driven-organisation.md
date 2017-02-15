---
layout: post
title: "The goal driven organisation"
date: 2017-02-15 09:00:00
author: jflowers
tags: [Agile, Goals, Delivery, Culture] 
---

**Quarterly team goals are an effective way to establish organisational purpose, direction and alignment while supporting team agility. But be vigilant - they can be used inappropriately.**

## Overview

Due to factors such as growth, acquisition, changing markets amongst others, organisations can find themselves in new environments to which they struggle to adapt.

Scaling Agile in these new environments is hard. The practices and tools that are frequently used to solve this problem can give the appearance of acting against team autonomy and agility. Teams and individuals naturally try to protect the engineering culture that they have worked hard to establish but by allowing new organisational needs to go unmet they put their autonomy and agility at risk.

In this post I will show how quarterly goal setting, using Agile principles and with one eye on the dysfunctions that can arise, is an effective way to meet organisational needs while protecting team agility and engineering culture.

## What needs are we trying to meet?

Senior leaders in an organisation need confidence that the creativity, intelligence, skills and knowledge of their employees are directed most effectively towards delivering long term business objectives. Ensuring that teams are aligned around those business objectives is traditionally called governance.

The top-down command-and-control connotations of that term do not sit comfortably in modern software development and rightly so. The responsibility for organisational alignment should rest with teams and individuals. 

A bottom-up approach to organisational alignment relies on transparency. Everyone in the business needs to know the business objectives and what all teams are doing to meet the objectives. Arguably the primary output of all product development and engineering teams is a statement of intent and a narrative of their progress.

Put another way, the first output of all knowledge work is shared, actionable knowledge - we can then decide how best to utilise that knowledge. Teams need to know that the decisions that they are making right now are the right decisions for the business. Without full transparency this is not possible. 

In medium to large organisations dependencies between teams are inevitable and I would argue not undesirable. Efforts to structure an organisation and system architecture in such a way that teams are able to work in isolation can reduce friction and time to market.

But at scale this becomes unrealistic without product, technology and knowledge fragmentation. Cross team collaboration can fail and cause unnecessary friction without organisational support and guidance. The ‘improvised collaboration’ that takes place daily and is essential for a business to innovate and succeed needs a framework of ‘scripted collaboration’ so that teams and individuals can assess the impact of requests from and to other teams, and avoid the false sense of urgency that arises without clear direction on the business priorities.


Teams and individuals also need a sense of how their work contributes to the larger whole. A sense of purpose and direction is intrinsically motivating. Similarly team autonomy and the space to make your own decisions are important aspects of any company culture that values its employees and wants to make the most of their talents.

However, they come under threat when a business lacks clear vision and priorities or the vision and priorities are not understood by everyone in the business.

### The learning organisation

By holding regular retrospectives and fostering a spirit of experimentation and continuous improvement teams have become adept at locking in their learning through process and system improvements. But how does an organisation as a whole achieve this?

If everything is experienced as change then nothing is. We’re just floundering in the zone of chaos. The organisation needs a lightweight means to capture its intent, key decisions, the evolution of plans and actual outcomes. Think of this as a thin slice or low-res snapshot of our current shared understanding. Once captured we can refer to it when evaluating the impact of change. Without this reference the organisation is reactive with little understanding of the impact of its decisions over the medium and long term. 

The tool that OpenTable has chosen to achieve organisational alignment, create a sense of purpose and direction and to act as a foundation to support learning is quarterly goals.

## Inappropriate use of goals

As with all tools, quarterly goals can be used inappropriately. Following Agile principles will help ensure that they are of benefit to the team and organisation but there is always scope for misinterpretation, and consequently there are approaches to goal setting that should be avoided.

### A pipeline of work

Goals should not be treated as a pipeline of engineering deliverables that the team are expected to deliver in priority order. Customer and stakeholder value should be the driving factor in defining goals. Simply clearing a backlog of fixed scope work items is an ineffective way to run a business such as OpenTable.

The project pipeline will be familiar to anyone who has spent time at a digital agency or production shop. By delivering a pipeline of work the agency has a guaranteed revenue stream as it is work that clients have agreed to pay for. A business such as OpenTable could successfully deliver a pipeline of work on time and within budget and still go out of business.

We should ask ourselves where do we want to be in three months time and, at this moment, what do we think we will need to do to get there. Contrast this with the approach whereby we select a number of work items from a backlog that we think we will be able to complete in three months. We should favour the first approach over the second.

Avoid thinking of goals as a pipeline of fixed, engineering deliverables.

### The arbitrary deadline as a means to push teams to deliver

The three month time horizon for goals is as good an arbitrary period of time as any other. It provides a constraint for thinking beyond the current sprint and considering the medium and longer terms. But we mustn’t lose sight of the fact that it is arbitrary. 

We should feel a sense of obligation to our colleagues to meet the commitments that we make to them. If we accept that all models and plans are wrong but are useful and allow us to move forward, then we should be rigorous in doing what we can to invalidate our plans as early as possible, adapt them and share this information with all stakeholders. 

Teams should not be pressured into meeting a goal at all costs simply because of the impending end of quarter. We should all agree that the level of effort and duration are important factors in determining business value and prioritisation but driving teams to meet arbitrary deadlines will not help them establish a regular, sustainable pace that is the foundation for reliable forecasting.

Conversely the element of time cannot be ignored and the cost of delay to the business of not having a feature in the hands of customers beyond a certain date is too great.

Avoid treating the end of the quarter as an arbitrary deadline.

### A low tolerance for risk and change

An aligned business should not be rigid to the point that it is inflexible to changing circumstances. If business objectives change, teams should be able to adapt and realign around those new objectives.

What we assumed was important at the beginning of the quarter is very likely to be proven not to be so as the quarter progresses. New information and feedback from stakeholders, development teams and customers may invalidate our assumptions. Fast, frequent feedback and actual outcomes should allow us to adapt our goals and business objectives.

Don’t allow goals to lower our tolerance for risk and change.

### Silo-ed behaviour and working against collaboration

_"It’s not in our goals"._

Once a team has agreed its goals for the quarter the temptation may be to push back on anything that is considered to fall outside of them. Only so much can be anticipated three months in advance and there will always be new requests and changing priorities.

A business that embraces change should allow for goals and business objectives to change as new information emerges. Requests that originate from outside the team should be considered in light of longer term business objectives and priorities. If this means adapting the team’s plans for the quarter then so be it if it is in the best interests of the business as a whole.

Avoid a team-first mentality.


## A web of goals

_"How does what I’m doing right now contribute to the success of the business?"_

This is the question that everyone in the organisation should be able to answer. It should be possible to map each story on a team’s task board to the team goal and from there to the longer term business goals. Longer term business goals can be broken down into shorter term interim goals thereby adding more levels to the hierarchy.

Taken together the business goals are the connecting threads that run through all team goals and provide the context that establishes team and organisational purpose and direction.

Below is an example of a goal hierarchy for the OpenTable Global Traveller initiative:

<pre>
> Global Traveller
	> Create a global habit
		> Global Platform feature parity
			> Add multilingual support to start page features.
</pre>

In this example there are two levels of business goals which are needed to define the context for the value that will be realised from delivering the specific team goal - in this case adding multilingual support.

By following the links up the goal hierarchy and then back down along alternate paths, new connections and opportunities for collaboration that may otherwise go unnoticed are revealed.

Here is another example:

<pre>
> Global Platform
	> Allow diners to book any restaurant on any domain in any language
		> Launch all restaurants from English speaking domains in Australia
			> Enable all English restaurant natural language URLs
			> Enable cross domain search results
		> Allow users to book in any language on OpenTable.com
			> AB test a language switcher on .com
			> Allow users to receive emails in their selected language
</pre>
			
The highlighted team goals are not dependent on each other but the connections established by the higher level business goals suggest that collaboration and coordination between the teams would be beneficial.

## Setting goals

### The team mission statement

When setting goals for the first time a good place to start for any team is to consider its purpose. Why do we exist? A team mission statement can provide a framework for setting goals that have value and meaning to the team and to the business.

As an example, here is a possible mission statement for a team responsible for transactional communications:

1. Maintain transactional communications systems in production so that the business has a reliable and performant transactional communications system, and customers receive timely and relevant communications.

2. Build transactional communications capabilities that increase value for our customers and help the business meet its objectives.

3. Evolve transactional communications systems in line with the company’s technical vision for the platform to position the business to best meet the evolving needs of its customers.

4. Foster a supportive and welcoming team culture that values innovation, collaboration and continuous learning.

Quarterly goals can then map back to elements of the mission statement:

1. Maintain transactional communications systems in production so that the business has a reliable and performant transactional communications system, and customers receive timely and relevant communications.  **Reduce the level of noise from alerts so that the team can more effectively respond to production incidents.**

2. Build transactional communications capabilities that increase value for our customers and helps the business meet its objectives.  **Launch a thank you email to reinforce the benefits of booking with OpenTable.**

3. Evolve transactional communications systems in line with the company’s technical vision for the platform to position the business to best meet the evolving needs of its customers.  **Migrate transactional communications systems to Apache Mesos.**

4. Foster a supportive and welcoming team culture that values innovation, collaboration and continuous learning.  **Create an engineer on-boarding playbook to ensure that new team members have a great first experience of the team and OpenTable.**

Everything that a team does should relate to a team goal and the mission statement. 

The thank you email goal example also sits within the web of goals as described above. An example of how it might relate to higher level business objectives is:

<pre>
> Increase the frequency of yearly active bookers
	> Improve the post dining experience
		<span style="font-weight:bold;">> Launch a thank you email to reinforce the benefits of booking with OpenTable.</span>
</pre>


### The anatomy of a goal

- **The goal summary** succinctly captures our intent.
- **The customer and business value** that this goal will unlock. This is the most important element of a goal as it defines why we think it is important.
- **Key metrics** that will tell us whether the goal has been achieved.
- **Steps to achieve** includes anything that we think is worth capturing now in terms of discrete aspects of the goal. They can include interim milestones that will give us early feedback on how we’re progressing towards the goal.

## Reviewing goals

How should teams incorporate goals into their regular planning practices?

Sprint reviews, demos, retrospectives and planning sessions give teams a chance to reflect on and share their progress and collectively agree their next steps. We want to aim to keep the goals process as lightweight as possible and incorporating a regular review of goals into existing agile rituals is preferable. 

Fifteen minutes spent reviewing a team’s goals and agreeing changes when needed every other sprint planning session adds an extra dimension to planning, and gives a team more confidence that they are making the right decisions for the coming sprint.

Goals sit alongside a number of other elements that can be incorporated into planning that help establish team situational awareness:

- **goals** provide purpose and direction and are adapted based on outcomes;
- **workflow metrics** provide insight into what we’ve done and how we’ve done it and suggest improvements that we can make to our processes and practices; and
- **maps** allow us to navigate a path towards our goals.

The data captured by these three elements can then be interpreted and enriched by the team to tell a story about the team’s progress and intent.

## The quarterly goal setting cycle

In the same way that the team comes together to reflect and plan on a regular cadence the quarterly goal setting cycle is when the organisation as a whole does the same.

### The roadmap

So that everyone in the organisation understands the vision and business objectives the quarterly goal setting process should be kicked off with the business reiterating the medium and long term objectives. A review of progress towards those objectives and any changes that have been made since the last goal setting process should be highlighted and explained. This helps establish a consistent narrative that is essential when attempting to align an organisation towards a shared vision. 

Too often a company roadmap is created and presented only to be discarded and forgotten within weeks. The same rigour combined with Agile principles applied to planning at the team level should be applied to the medium and long term business objectives.

As I have shown these are the connecting threads that run through all team goals and establish organisational purpose and context. Left to fray, tangle and rot organisational alignment is impossible.

Once the business objectives have been firmly established teams and individuals are better placed to make the right prioritisation decisions.

### Team dependencies

The goal setting process provides the opportunity for the ‘scripted collaboration’ mentioned earlier that is needed to support and protect the daily ‘improvised collaboration’. When teams have agreed the things that they need to do to help the business best meet its objectives, dependencies on other teams should be called out.

Teams can then review all requests for collaboration, discuss with the relevant team and reach a shared understanding of what is expected, how they will work together and the impact on their other goals.

### The problem of local optimisations

It is unrealistic to expect members of a team to hold a complete model of the business in their heads which they can refer to when making decisions. Teams need the mental space to be able to focus on the domain and part of the system for which they have direct responsibility.

The downside to this is known as the problem of local optimisations. Over time small decisions made without taking the whole business into account accumulate. We need a means to either anticipate them or to realign on a regular basis. 

The quarterly goal setting cycle and the goal reviews give us a means to achieve this. They act as checkpoints to allow the organisation to realign.

### The organisation’s statement of intent

Following a final review with senior leadership to validate that the team’s interpretation of the business vision and objectives is aligned; and to make the difficult prioritisation decisions that it has not been possible to reconcile at the team level, the outcome of the goal setting process can be seen as the organisation’s statement of intent.

The artifact that is produced as a result of this process is the sum total of all business objectives, team goals, team dependencies and connections. For full transparency and as a reference point for ongoing decision making and for understanding the impact of change this artifact should be made accessible to everyone in the organisation. 

## Conclusion

Quarterly goal setting establishes the organisational superstructure within which a valued product development and engineering culture can flourish.

We often make the mistake to assume that an approach that values adaptive planning, team autonomy and less top-down control has less rigour and accountability. In fact the opposite is true. Applying Agile principles at scale is hard and relies on skills that may not come naturally to everyone in the business. But done well it results in:

- **more commitment** - to each other and the business;
- **more accountability** - to each other and the business;
- **more rigour** - in validating assumptions and making sure we are doing the right thing;
- **more planning** - done continuously rather than upfront; and
- **more connection** between individuals, teams and the business vision.

And by meeting the needs of senior management and giving them confidence that the energies of everyone in the business are directed most effectively towards business objectives team agility is protected and supported.
