---
title: "One Million Recordings at Squeaky"
date: 2022-06-13T19:10:21+01:00
draft: false
tags:
  - DevOps
  - Software Engineering
---

## What is Squeaky?

[Squeaky](https://squeaky.ai) is a web application that offers a suite of analytics tools for companies that includes session recording, analytics, NPS scores, sentiments, user journeys, events tracking and more. Myself and [Chris Pattison](https://www.linkedin.com/in/pattisonchris) have been building Squeaky over the last 6 months or so, and we launched recently in April. Squeaky competes with other tools such as [HotJar](http://hotjar.com) and [PostHog](http://posthog.com), however we are massively centered around privacy and making sure that users remain annoymous.

## The tech stack

Squeaky's tech stack has been largely unchanged since the MVP. The main components are:

1. `web` - A Next.js front end that powers the public website. It's written entirely in TypeScript and can be ran as a server, or exported as static HTML.
2. `app` - A Next.js app that powers the entire logged in experience. It's also written entirely in TypeScript, and communicates with the rest of the stack using GraphQL.
3. `api` - A Ruby on Rails app that handles 99% of the backend for Squeaky. It handles the sessions, the GraphQL server, all of the background jobs, and nearly all of the communication to the database.
4. `gateway` - A lightweight Go app that handles the websocket connections to ingest all of the data into Squeaky. It's much more effective at this than the Ruby app.
5. `script` - The code that is loaded onto customers websites to stream the data into Squeaky. It's written in TypeScript.
6. `terraform` - The Terraform repo that provisions 100% of the stack.

All of the apps run on Fargate in AWS. We use Postgres as the primary database in RDS, and ElastiCache for Redis. All of our assets, including the script that is loaded on customers websites, are served from CloudFront. Our entire CI/CD pipeline runs in CodeBuild, and our logging and monitoring is all done with CloudWatch. We send all our transactional emails with SES.

I'll try and answer some of the questions that people probably have:

#### Why build a website with React?

React is really good at composing stuff, and Next.js renders it to HTML anyway. 

#### Isn't Ruby old and slow and dead?

It's doing great, and you can move very fast with it. The bottleneck for a product like this is always going to be pulling the massive amount of data out the database, so Ruby's speed (or lack of) is negligible.

#### Why write one single service in Go?

It's really fast and uses barely any resources compared to it's Ruby equivalent.


## How data gets in

The actual ingest path is relatively simple, but it's also the scariest and most fragile part. 

When a visitor lands on the site, the script is fetched from the CDN. When that loads, a request is made to fetch a configuration object from the API. This configuration is used to enable or disable certain features such as NPS, Sentiment or the Magic Erasure. After this, a Websocket connection is opened to the gateway. A few checks are made at the start, such as an origin check. If all is well, we begin streaming in all the events required to play back a session, such as a tree markup of the dom, and mouse and scroll positions.

![A rough guide of the ingest pipeline for Squeaky](/images/ingest-stack.png "A rough guide of the ingest pipeline for Squeaky.")

We send this information every 100ms, which causes some problems for us on the backend. For one, we can't just write directly to the database at this rate. If we had a handful of users on site it might be okay, but even at our current size it is impossible.

Second of all, the events data is massive. Most people don't optimise their sites for performance, and as such we end up ingesting massive DOM markups from Wordpress, as well as megabytes of unnecessary Tailwind stylesheets. The solution that we currently use is to `zlib` to compress the data on the way in, and write it into a Redis list as a base64 string. This is considerably more effecient to store than a regular JSON string. We use Redis to buffer the writes as it is more than capable of handling the throughput. Go is also more than capable of compressing this data in realtime as it is streamed in.

Another issue that we have is that we're never quite sure when a visitor has left for good. Because of this, we leave the data in [Redis](https://redis.io) for 30 minutes, resetting the TTL after every disconnect. This way, we give them enough time to be sure they are truly gone, and not just changing pages (as you can't keep sockets open between pages). We put a delayed job on a [Sidekiq](https://sidekiq.org) queue to trigger the saving process after 30 minutes.

When the 30 minutes is up, a Ruby background worker picks up the message and pulls the data out of Redis, decompresses it and performs some validation before storing it. A great deal of things happen during this part of the process which I won't go into.

## Data growing pains

At the time of writing, Squeaky has processed a little over 1,000,000 recordings, which includes about 1,000,000,000 events. All of these events are stored in Postgres and can include anything from click coordinates to entire pages of HTML and CSS (in a tree format). Our RDS snapshots are around 1TB, but I'm not entirely sure how large the database itself is.

As the database grows we continue to find queries that are no longer fast enough. It's been a battle trying to optimise them where possible, or to change the way it is stored to be able to extract them more easilly. It's often no longer possible to run queries at all that do not have an index, and even then I need to be careful. For example, I tried to find out if Squeaky had any errors at all over the last week, so I hopped into the Rails console and ran the following:

```
Site
  .find(site_id)
  .events
  .where(
    'timestamp >= ? AND event_type = ?',
    Time.now - 1.week,
    Event::Types::Error
  )
```

This immediately consumed one full CPU core and I had to go and find the PID inside Postgres and kill it.

## What I think I got right

The number one thing I think I got right is picking boring tech that I knew well. It was very tempting to use a Serverless stack with Kinesis and have some fancy pipeline, but in the end I'm glad I went with boring Redis, Postgres and a basic Rails app. There have been many occasions where something has caught fire and I've been able to jump straight into the Rails console and get it sorted.

I also stand by React for building every type of front end. It's so composible, and the tooling you get with Next.js is amazing! People on HackerNews will bash it and say that everything on the internet should be written in HTML4 with no CSS or JS, but I find this approach to work perfectly.

I decided early on that I would use as little packages as possible. Our front ends have about 15 NPM packages between them, excluding types. The entire component library was built by me and uses no dependencies whatsoever. All the interactive stuff such as accordions and carousells are also all hand built. This has kept our page weight down to an absolute minimum and has been no trouble at all to keep up to date. The API also only has around 10 gems in total. I constantly keep all apps on the latest stable versions of their languages, frameworks and dependencies.

## What I think I got wrong

Although I just said that Postgres was something I got right, it's also something that I got wrong. It's not designed to run analytics queries against billions of rows, and I should have noticed this earlier and switched to a more appropriate store. We're investigating [ClickHouse](http://clickhouse.com) as an alternative now. I also think in retrospect I should have built the entire thing publicly and accepted help from the community. This is my plan long term, but there's a lot to unpick now so that it would work for others if they chose to run it themselves.

I've also kept my day job whilst building Squeaky so nearly all of it's development happens in the evenings and weekends. I wish I'd taken some more time off in between leaving WonderBill and starting at Deliveroo so that I could have focussed on it better.

## Looking fowards

Squeaky is still in it's infancy but it is already competing with VC backed giants with hundreds of engineers. Chris and I have built something that I think it's crazy impressive for just two people. We're starting to roll out some seriously impressive features and there are many more in the pipeline. The tech is also maturing and becoming very stable. All of our metrics are trending in the right direction and I think Squeaky will be a huge force in the new, privacy-oriented web.
