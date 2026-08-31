# Channels

Entries carry structure. Current rules come from step 7's research, not from here.

## Repo types

- **Library or framework**: stars come from people who use it, or expect to. Ecosystem
  surfaces and long-form matter most.
- **Application or service**: stars come from people who run it. Demos, screenshots, and
  self-hosting communities.
- **CLI or TUI tool**: stars come from a demo GIF. Aggregators reward these more than any
  other type.
- **Learning resource, reference implementation, showcase, or example**: stars are
  bookmarks. Curated lists and the canonical index for the topic are the whole game. A
  repo whose owner calls it a portfolio piece or a showcase is this type.
- **Template or boilerplate**: stars come from people starting a project today. Registry
  and template surfaces, search-driven.
- **Curated list or dataset**: stars come from breadth. Aggregators once, then inbound
  links forever.
- **Plugin or extension**: stars come through the host's marketplace, not through GitHub.
- **Personal (dotfiles, configs, experiments)**: no realistic star path. Say so at the
  ceiling step rather than building a campaign. No channel below applies, whatever its own
  fit line says.

## Aggregators

One-shot, high variance, and spent permanently. The page must convert before any of these
run.

**Hacker News**. Good for tools and services with a clear novel angle, and for writing
that happens to have a repo attached. Punishes marketing voice, launch framing without
substance, and anything that reads as a press release. One-shot in practice: a
resubmission after a failed launch rarely lands. Fits CLI tools, applications, libraries
with a genuinely new idea. Write a plain title, no adjectives, no "Show HN" unless it is
your own work and it runs. Post the write-up rather than the repo when there is one, be
present in the comments for the first two hours, and answer criticism directly.

**Lobsters**. Smaller, more technical, invite-only to post. Punishes self-promotion
without participation history. Fits libraries and deep technical work. Tag accurately, and
expect the comments to be more expert than HN's.

**Subreddits**. Good for reaching a specific stack's practitioners. Punishes
self-promotion, hard, and the rules differ per subreddit and change. Read the sidebar and
recent removals before drafting. Phase it as one-shot: a second post to the same subreddit
works only with genuinely new content and months in between. Fits every type. Write as a
person sharing work, lead with what you learned or built rather than a link, and answer
every comment.

**Dev.to, Hashnode, Lemmy, and similar**. Low ceiling, low risk, repeatable. Good for the
long-form piece that feeds the other channels. Fits every type. Cross-post with a
canonical link back to your own blog.

## Curated lists

Permanent placements, repeatable across many lists, and the highest-return work for
learning resources and reference implementations. Do these before any one-shot channel.

**awesome-\* lists**. Good for steady inbound over years. Punishes PRs that ignore the
contribution rules, which are usually specific about ordering, formatting, and minimum
quality bars. Fits every type. Read each `CONTRIBUTING`, and open one PR per list matching
the existing entry format exactly. Step 7 bounds the search.

**The topic's canonical index**. Many niches have one authoritative list owned by the
spec, the framework, or the standard. Being listed there outperforms every aggregator for
a reference implementation. Find who maintains it and what it takes to be listed. Check
whether the repo is already on it before planning to get on it, and if it is, the work
becomes optimising the entry: every field filled, every facet used, so the repo appears in
the filters visitors actually apply.

**Language and framework directories**. Ecosystem sites that catalogue projects for a
stack. Slower, permanent, and usually a form submission rather than a PR.

## Ecosystem surfaces

Repeatable, permanent, and reached by people already searching for the problem. Underused
relative to their return.

**Package registries** (npm, PyPI, crates.io, Maven Central, Go pkg site). The registry
page is often seen more than the repo. Make sure it renders the README, links back, and
carries the same pitch.

**Container and image registries**. Docker Hub and GHCR pages are landing pages. Fill in
the description and the usage block.

**Marketplaces** (VS Code, JetBrains, GitHub Actions, browser stores). For plugins and
extensions this is the only channel that matters. Listing quality drives installs,
installs drive stars.

## Direct community

Repeatable, low volume, high conversion. Slow, and it compounds.

**Discord and Slack servers, forums, mailing lists** for the stack. Good for reaching
people mid-problem. Punishes drive-by links from accounts with no history. Fits every
type. Participate first, share when it answers someone's actual question.

**Stack Overflow and equivalents**. Answering questions in the niche with a working
answer, mentioning the project only where it genuinely is the answer. Slow, permanent, and
it ranks in search.

## Long-form

Repeatable, and it feeds every other channel. Where the intake says the user will write,
this returns more per hour than anything else on the list.

**Own blog with canonical link**. Good for the technical story behind the repo: a
decision, a benchmark, a migration, a thing that broke. Fits every type. Write about the
problem rather than the project, and let the repo be the thing the reader wants next.

**Guest posts and ecosystem blogs**. Framework and vendor blogs take contributed posts and
carry far more reach than a personal blog. Ask.

**Newsletters**. Some take submissions, and for those, send the write-up rather than the
repo. The ones with the most reach in an ecosystem are usually editor-curated with no
submission path at all. Those are not a channel action: record them as a downstream metric
on the article that might get picked up, and note what their editors habitually link.

**Ecosystem publications with open author programs**. Some ecosystems run a community
publication that grants publishing rights to anyone who asks and participates, rather than
commissioning writers. That means front-page placement on a site the ecosystem already
reads, with no following required, and cross-posting with a canonical link is normally
supported. Find the ecosystem's equivalent during research, and get the credentials before
there is anything to publish, because the process runs on human response times.

## Video and social

Repeatable, and the only channel where a demo outperforms an explanation.

**Short demo clips** (30 to 60 seconds, no narration needed). Good for CLI tools and
anything visual. The same clip serves X, Mastodon, LinkedIn, and the README.

**YouTube**. Higher effort, long tail, worth it only when the intake says video.

**X, Mastodon, LinkedIn, Bluesky**. Reach depends almost entirely on the existing
following recorded at intake. With no following, these are a place to park the demo clip
rather than a distribution channel. Write one specific claim plus the clip, not a thread
of build-in-public narration.

## Contribution-driven reach

Repeatable, slow, and it builds the credibility every other channel checks.

**Upstream PRs and docs contributions** in the projects the repo depends on. Good for
becoming a name people recognise before you ask for attention. Fits libraries and
reference implementations most.

**Being the answer in someone else's issue tracker**. When a project's issues repeatedly
ask for what this repo does, a helpful comment is legitimate distribution. Once per
thread, never templated.
