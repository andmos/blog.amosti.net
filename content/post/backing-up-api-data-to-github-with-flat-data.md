+++
author = "Andreas Mosti"
date = 2022-01-17T18:16:18Z
description = ""
draft = false
slug = "backing-up-api-data-to-github-with-flat-data"
title = "Backing up API data to GitHub with Flat-Data"

+++


For the last 10 years or so I have used Trello as my preferred service to keep track of my reading.

I [wrote a blog post](https://blog.amosti.net/how-i-read/) about this setup and my technique with regards to reading, which, upon re-reading, I see that both my technique for reading and writing-skills has improve, so that's that.

In 2017 [Atlassian acquired Trello](https://techcrunch.com/2017/01/09/atlassian-acquires-trello/?guccounter=1&guce_referrer=aHR0cHM6Ly93d3cuZ29vZ2xlLmNvbS8&guce_referrer_sig=AQAAAB4EB-u4iDYnVbIaNgQ5Zc3vh2a4oO12KL7tXvnrIAjHpVo76z6XqXP_F2I7-TDusMGfPvzMr21IRMIyUbenmxznHq_Ap4RcAzuG0yRLxve7Kc8naDdpLEgavPQkU7wuxfOIet-bTDAHjct5eB8DZujMtQiVKaZ2JSq1ji7v7Ypy), and safe to say, thing are starting to become more "enterprisey" over there.

I figured I won't be staying with Trello forever, and the amount of sunk-cost I have compounded in my simple reading list is astonishing. After some grooming, the backlog has currently 242 items and my done list has 450 entries. 

What can I say, I like to read.

Some years ago I began my humble planning for moving away from Trello by writing a [wrapper API service](https://github.com/andmos/ReadingList) for the reading list,
using the Trello API itself as repository layer, so the switch to some other service could be easier. Maybe this API will act on top of something like SQLite or Firebase, who knows. To make sure I always had a copy of the data from Trello, I wrote a small bash-script with some `curl` calls and set up a cronjob. Now cronjobs are messy, I need to remember that I have it running, correct it if it runs into some error, and - after all - back up the data that comes from it.

Searching for a better solution than bash and cronjobs, the next step on this pet project of mine came February 2021, when GitHub releases a cool project called [Flat-Data](https://next.github.com/projects/flat-data).

In the projects own words:

> Flat Data is a GitHub action which makes it easy to fetch data and commit it to your repository as flatfiles. The action is intended to be run on a schedule, retrieving data from any supported target and creating a commit if there is any change to the fetched data.

So in a nutshell, Flat-Data allows us to set up a simple GitHub Action, point it at some data source (like an API or SQL database), query the data, run some processing on it, and have it checked in to Git. A great tool for the data scientists I would
imagine, or for a guy who just want to back up his reading list from an over-engineered API layer on top of Trello.

It's easy to get started. All we need is a repository and a GitHub Action. 
My repo is [ReadingList-Data](https://github.com/andmos/ReadingList-Data) and contains a few files: `backlog.json`, `done.json`, `reading.json` and `postprocess.js`.

The GitHub Action itself using Flat-Data looks like this:

<script src="https://gist.github.com/andmos/00fe8179aac87a805e2d4d12a749058e.js"></script>

Every Monday afternoon this action runs, fetches the current status from the reading list API, runs it through the post-processor
script (a simple Deno script that only formats the JSON) and creates commits with the changes. It could not be simpler.

As a bonus the commit history works as a great timeline to see when a book was added to the backlog or I was done reading it. 

![Commit-history](https://user-images.githubusercontent.com/1283556/149825223-9894be37-ff14-4788-92dd-e3eb654e06cd.png)

This is a quite simple and banal use case for Flat-Data, but hopefully it will inspire someone out there.