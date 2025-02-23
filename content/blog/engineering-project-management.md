---
layout: post
title: Engineering project management
date: 2025-02-21
tags: [github, engineering]
---

## Introduction

As a Senior Engineer, one of my duties is breaking down large pieces of work into ordered and prioritised set of tasks, each scoped appropriately, ideally sized no more than 1 - 3 days.

It's unlikely each task initially discussed during a planning session will be part of the final set, nor is it likely they appear in the initial ordering. As sessions progress, tasks may need to be re-prioritised, dependencies between tasks may get raised, and the scope of existing tasks may grow or shrink as newer tasks are discussed. It's great to have a quick workflow which deals with these inevitable changes as it will result in more productive sessions.

I found working in a text box driven GUI slow as I had to repetitively open dialogs, move my mouse between text boxes, type words, and submit.

I was looking for a way to work smarter, do less tedious project management manual processes, and focus on solving real engineering problems.

For these reasons I made [scrumfaster](https://github.com/itsjfx/scrumfaster), which allows you to create GitHub issues and project items with Markdown.

The initial inspiration of using Markdown (other than it being great) came from an initial planning session of a project where my client had no project board set up, so we wrote a high level plan of the next few months in markdown. The client did not want to pay for or set up a Jira instance, so we decided to trial GitHub projects.

[scrumfaster](https://github.com/itsjfx/scrumfaster) was initially a disgusting 50-line Bash script I wrote on a Friday night to get the tasks from that markdown doc into GitHub. From there we continued using it to create new sprints and bootstrap future projects. I decided it needed a rewrite in Python, which would allow more advanced features and less bugs, so I did that and published it online.

## A showcase

Below is an example of some markdown which `scrumfaster` will accept (also referenced in the repo):

```markdown
# my cool board

## Sprint 1

* [ ] Profile avatars
    * [ ] Create database migration for avatar field [@itsjfx] [status=Done] [labels=database] [1]
        * Name the field `avatar` in the `users` table
        * Set value for existing users to https://...
    * [ ] Accept avatar parameter in `update_user` API call [@itsjfx] [labels=api] [1]
        * Use existing image upload mechanisms
        * Limit image size to 10mb
    * [ ] Display and allow updating avatars on frontend [labels=frontend] [2]
        * Only display avatars on the users public profile page
        * Thumbnails aside comments to be implemented in later card
* [ ] Dark mode
    * [ ] Add ui toggle for dark mode [labels=frontend] [1]
        * Store preference in local storage
    * [ ] Implement styles [labels=frontend] [2]
        * Apply styles dynamically based on user preference
* [ ] Delete jeff from database [labels=database] [1]
```

Some key notes:
* the nested tasks are syntactic sugar to create a task for each of the lowest children which the parent tasks as prefixes
    * e.g. `Profile avatars: Create database migration for avatar field` would be the first task created
* you can specify assignees, labels, points, and status in-line of the card
* milestones can be specified as markdown headings
* you can create GitHub issues, or GitHub project draft items

The result is a natural and readable markdown representation of a project.

There's much more information on the repository, so if this got your interest please check it out: <https://github.com/itsjfx/scrumfaster>

## Always be an Engineer

The main takeaway of this post is not the cool project I wrote, but that engineering should span across everything you do. Sadly, as you get promoted, you will find yourself getting more duties, and doing less engineering work. You'll be responsible for people, be more involved in your companies politics, and spend a lot more time in meetings.

I believe a good engineer will be able to find ways to make their non-technical or tedious work exciting by identifying clear gaps and iteratively creating a solution which makes them more productive. You may be able to complete additional technical work or other important non-technical tasks as a result. More importantly, you will continue practising your engineering skills and demonstrate your competency to your peers. Never lose the engineer within you as you become more senior, but you must not neglect your key duties.

## See also

* <https://github.com/mheap/markdown-to-jira>
* [Are we losing the engineering part of software engineering?](https://www.youtube.com/watch?v=_E0IBNvAXX8)
