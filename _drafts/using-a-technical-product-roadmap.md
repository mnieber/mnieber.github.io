---
layout: post
title: "Using a Technical Product Roadmap to create team alignment"
date: 2020-28-10 16:21:59 +0100
categories: project management
---

## Introduction

To effectively work together as a software development team it is critically important
that the team is aligned on the product vision and planning, both in the short term
and in the medium term. This includes:

- understanding the user requirements so that team members can agree on what the product tries
  to achieve
- sharing an ubiqitous language so that key terms have the same meaning to all team members
- having a shared understanding of the product development plan, so that team members do not
  work at cross purposes

Almost all companies I have worked with struggle with creating this alignment. Teams often rely on the
Product Backlog to create a shared understanding of the upcoming steps, but as I explain in the next section
this approach fails to produce good results. As an alternative, I will introduce a particular type of
roadmap document, including a small example, and discuss how it can aid team alignment and planning.

## Why using a Product Backlog fails to create team alignment

The main reasons why a Product Backlog fails to create team alignment are:

- it's tedious to read. The usual way to present written information is as a narrative, where text further
  down builds on the text that preceeds it. Because Product Backlog stories are meant to be self-contained
  they are repetitive, and there is no natural flow leading from one story to the other. Moreover, the reader
  has to open and close the stories to read them. All of this makes reading backlog stories unpleasant and
  unrewarding.

- it lacks structure. Maybe the top will be ordered, but the rest will have an almost random order.
  That rest will contain both interesting and important stories and quick braindumps that are basically noise.
  And again, since the stories are meant to be self-contained, it's hard to see how they are related. It's like
  trying to understand a picture by looking at one puzzle piece at the time, and having to guess along the way
  how the pieces fit together.

- it lacks a user perspective. While it's usually possible to imagine how a story helps the user to achieve their
  goals, it's hard to reconstruct an entire user workflow from reading backlog stories. And when it comes to user
  workflows, the devil is in the details. User workflows tend to change in important ways once you start to work on
  getting the details right. Figuring these details out sooner rather then later helps to align the team on the
  development plan.

As a result, people will not want to read the backlog, and when they do, they will struggle to obtain
an impression of what the team will be working on in the next few months, and where the product will be.
Usually, documents that describe the short and medium term plans exist alongside the backlog, but it tends
to be unclear how these plans relate to the question of what the team will do in the next months.

In the end, it often turns out that the actual plan exists mostly in the heads of the team-members,
and it's communicated by talking. The biggest drawback of this situation is that it never becomes clear
which tradeoffs are made between the different options for improving the product. When information
about possible next steps is scattered then weighing the options quickly becomes complex and overwhelming.
As a result, nobody will feel responsible for doing this important work, and when it's time to do the sprint
planning, some pragmatic ad-hoc choices will be made. This situation leads to poor planning choices.
Moreover, the lack of accountability for deciding the plan means that the team will not be able to
learn from past planning mistakes.

## What is a Technical Product Roadmap?

By a Technical Product Roadmap I mean a document with the following properties:

- it contains milestones that describe a user workflow with enough product detail to clarify the user
  experience. It should also have enough technical key details to give some guidance and constraints
  regarding the implementation
- each milestone typically describes a few weeks of work and fits an A4 paper. It should be a concise
  and easy to read text that may link to other documents that give more details.
- a milestone focusses on user workflows but may also describe purely technical tasks
- the roadmap contains a glossary that explains all the key terms (this helps to keep milestone descriptions
  compact)
- upcoming milestones should be precise. This mean they should not have obvious gaps in explaining how the
  product will behave or leave important implementation questions unanswered. On the other hand, milestones
  that are further in the future will be much more sketchy.
- it should be an authoritative document, in the sense that there are no parallel documents that are also
  a source of truth for what the team will be working on.

The first point is the most important one. A technical product roadmap focuses on a desired user experience,
and outlines a technical approach for implementing it. It should not try to answer every question related
to user experience and implementation. Instead, it should have just enough information to give the reader an
impression of how the product will behave and how that behaviour will be implemented. As stated, where
appropriate it can and should refer to other documents for more detailed technical discussions.

The last point is also important, because otherwise there will be no real transparancy about priorities.

## An example of a milestone

I will use the example of a website that allows user to share lists of dance move videos.
The website can be used by people who want to learn how to dance. One of the product requirements is
that some videos should not be made available to all users. This is because some dance teachers make
videos available to their students but do not want these videos to spread to the whole internet.
The example milestone below is smaller than usual for reasons of brevity, but it shows the idea.

### Glossary

- **Move** - the combination of a description and a video that explain a dance move to a user
- **Move Author** - the user who created the Move
- **Private Move** - a Move that is only visible to the Move Author
- **Public Move** - a Move that is visible to anyone. Public Moves must have a Public Video.
- **Public Video (Url)** - a video url that has no sharing restrictions.
- **Regular User** - a user who is not an Uploader
- **Uploader** - a person who is allowed to add Public Videos. Uploaders will contact the Video Owner
  to make sure that the video is indeed freely shareable.
- **Video Owner** - the person who produced/owns the video

### Milestone 3: Users can create public and private moves

1. A user can log into the website
   - 1.1 If the user is not yet an uploader, then they can send a request to become one
   - 1.2 A user can log in from different computers but no more than twice (to prevent a group of
     people from sharing one account)
2. Uploaders and Regular Users can create a new private move
   - 2.1 A move will have a title, description, video url and tags.
   - 2.2 The move will have a url that is only accessible to the logged in move author.
3. An uploader can create a new public move
   - 3.1 The uploader must confirm that its associated video is indeed freely shareable
4. A regular user can create a new public move if they use an already public video
   - 4.1 If the uploader later reverts the decision to make the video public then the move
     automatically becomes private. If the video goes back to being public, then so does the move.
5. The backend does not use data mutation to change public videos/moves into private ones.
   - 5.1 Instead it stores domain events from which the status (public or private) can be reconstructed.
   - 5.2 An administrator can view an audit trail that shows how the status changed over time

## Notes on the example milestone

As you can see, the glossary already explains several things about the application. It introduces important
concepts, and to a certain extent, the behaviour of the application follows from these concepts.

Ideally, every step in a milestone contributes to some overarching goal that is summed up in the milestone title.
Usually a milestone describes a viable version of a workflow, but not the ultimate version. In future milestones
the workflow may be updated and refined. For example, a future milestone can state that move titles are rejected
if they contain certain disallowed words.

Point 1.1 of the milestone illustrates how certain details are left out: it doesn't say how the user
can send a request to become an uploader. Maybe there is a contact form, or maybe there is an email address.
Since this information is left out we can conclude that this topic is apperently not interesting enough to detail it here.
Similarly, in point 2.1 it's not stated which fields are required and which are optional, because this is not
essential information for the roadmap. If a user story is added to a Scrum sprint for this feature then it probably
will specify this information.

In point 3 we don't repeat the fields of a move, because that is already mentioned in point 2 and we expect the
reader to be reading the whole milestone. This is an important difference to a backlog with user stories that must be
more self-contained.

Point 5 shows an example of mixing implementation details (5.1) and features (5.2). In my opinion it's important to
add key implementation decisions to the roadmap because the whole team should be on the same page about them.
If the decision to use immutable data and domain events is controversial then it helps to make this information visible
in a place where people can discuss this and sign-off on it. Without this information you cannot claim that the team is
aligned on the plan. Note that it's enough to outline the technical approach, the details can be described in other
documents.

## How a Technical Product Roadmap creates team alignment

Combining the user workflows and implementation outline in one document gives the reader a complete high-level
picture. If the entire development team understands and agrees on a milestone, then chances are good that
they are aligned. The milestone will not describe all the details, and therefore people may have slightly divergent
ideas, but they will agree on the main points. If during development some major changes to the plan occur then the
milestone description will become the common source of truth for the new plan. People can comment on the milestone
and propose changes, which leads to a discussion and eventually to an updated milestone description. Ideally, these
changes will not mean that the goal of an ongoing sprint also changes, just that it's achieved in a somewhat
different way.

The technical product roadmap invites you to think things through on paper (the value of which is often
underestimated, in my opinion). This is less agile than what Scrum or Kanban advocates, but it does leave
room for figuring things out during development. In fact, this is necessary to keep the milestone descriptions
concise and readable. Interweaving the description of the user experience with key implementation details
helps to ensure that the implementation choices are grounded in the user requirements. This is
important because surprisingly often the implementation is driven by what is technically possible rather than
following the most effective way to create the user experience.

Finally, the inclusion of a glossary helps a lot in creating a common understanding in the team about the main
domain and implementation concepts. Creating the glossary will be a process of exploration and discussion
that can clarify a lot of misunderstandings in the team.

## How a Technical Product Roadmap affects the planning process

Since the Technical Product Roadmap is the single source of truth for the product development priorities, it creates
transparency about the concrete plan for the next months. Of course, the number of options for improving the product
can still be daunting, but at least the current plan (even when it's constantly being debated and updated) is visible.
It's very important though that the milestones are concise and interesting to read so that people are inclined to
consult the roadmap often. When this is the case it fosters healthy discussion and good decision making because any
insertion of new work into the roadmap immediately makes it clear how this will delay other work. If the roadmap is
not an attractive document, it will be ignored and useless.

I personally do not attach deadlines to milestones. Rather than as a planning tool I use these roadmaps as a
communication tool, one that allows the team to be on the same page regarding priorities, features and implementation.
However, I do insert important dates (e.g. a fixed release date) into the roadmap so that it becomes clear which
milestones should be finished before these dates.

It's useful to combine the roadmap with Scrum or Kanban, where user stories are derived from the roadmap. The roadmap
leaves out steps that are not interesting for readers who want to understand the high-level plan, but a sprint planning
needs to be a complete description of the required work and should definitely include stories for them.

## Drawbacks of using a Technical Product Roadmap

The biggest drawback of a technical product roadmap is that it requires you to read. The amount of text is not
overwhelming, but you can expect to need 5 to 10 minutes to read through a milestone. When the roadmap is used
correctly, it will become a living document that people return to on a daily basis. In that light the required
time investment is modest, but still the fact that the roadmap is entirely text-based can be off-putting to people.

Another potential drawback is that creating a good roadmap requires skill. It's especially an art to choose the right
level of detail.

A roadmap works well when there are features that need to be developed, because these features can be described
as a work-flow, and by understanding and discussing work-flows the team can create a shared product vision.
The situation is different when most of the work consists of bug fixes and chores, because then the
milestones will not have an overarching goal. In this case the roadmap is still useful for clarifying the priorities
in fixing the issues, but it will in fact look much like a product backlog.

An alternative apprach that is less text-based is to use Story Mapping. Since this approach makes it easy to layout
the different stories in development iterations it can give a lot of insight about the product development priorities.
On the other hand, it provides a more loose description of user workflows and therefore gives fewer guarantees about
achieving team alignment on the development plan.
