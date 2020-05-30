+++
title = "What I want from a code review tool"

aliases = ["/2018/06/what-i-want-from-code-review-tool.html"]
+++

Many of my friends and colleagues will have heard me decry the state of code review tooling in the open source world. Here, I try to collect my thoughts on what I feel is missing; maybe it will help motivate some of the big players in that direction, maybe it will serve as a roadmap for my next side-project.

I'm going to assume that as a reader, you're bought into the general idea of code review: a place for people to get constructive feedback about a change they want to make (a "Pull Request") before that change is merged onto a target branch.

So what do I want from a code review tool?

**As a code author**, I want to easily see the status of my change. What needs to happen before I'm comfortable merging it? Some examples:

1. Have my reviewers looked at the code yet?
   * If not, I probably don't want to merge my code (obviously, there are exceptions).
   * There is some nuance here - as a code author, am I just looking for someone to look over my code, just to sanity check it? Am I looking for Janet to review this code, because we had a discussion about the design, and she's best placed? Do I also want Sophie to also take a look, because she probably knows a better language construct I could be using? Whose review is sufficient?
   * There's more nuance that gets added from the reviewer side; I've yet to use a code review tool which has relationships between reviewers: "I'm ok with it if Sophie is". It's hard to express these concepts elegantly, but I suspect it can be done.
1. Have any of my reviewers left comments which I haven't addressed? Which comments still need addressing?
   * My general rule of thumb is that all comments should either be addressed in code, or explained as to why they're not.
   * There is some nuance here - some comments are trivially addressable; some may need discussion, or I may try to address them, but not do a great job of it, and need follow-up. Ideally, the nuance of "I've left a comment, feel free to ignore it" vs "I've left a comment, I'm sure you'll address it fine" vs "I've left a comment, but would like to have another look before it's considered addressed" would be captured, in a low-friction way.
1. If any tests have been run, have any failed? If so, I probably want to investigate that before I merge my code.

I've yet to use a code review tool in the open source world which actually meets these three criteria. Most shy away from modelling the social interactions and connotations of code review (things like "I'm ok with it if Sophie is"), and instead try to dump all of the information that could be relevant at you. Saying in one place "Janet has accepted, Sophie hasn't reviewed yet", and in another "Janet mentioned half-way through a paragraph that Sophie should probably take a look). Worse - many of them hide information they deem out of date. When nuance can only be expressed through text, it's great to have the text there for people to read, but wouldn't it be nice if a green tick meant "Good to merge", rather than maybe "Good to merge" or maybe "Good to merge if you make this change" or maybe "Good to merge but it would be great for Sophie to have a look too"?

**As a code reviewer**, I want to see what I need to do to unblock people.

1. Has someone asked for my review, which I haven't provided?
1. Has someone responded to one of my comments?
1. Has someone acted on one of my comments where there's follow-up I should do?

Within the tool, there are two very distinct conversations to have; "Is this change generally sensible" and "Are the specifics of the code sensible?" - fortunately, the existing tools seem to cover this pretty well, with "change comments" and "line comments".

Now that the general concepts are covered, what specifics do I want in a code review tool?
1. High quality code browsing.
   * Syntax highlighting.
   * Click-through and find-usages of symbols.
1. High quality diffing.
   * Code diff chunks should include the name of the function, and type, the code is in.
   * Reviewing similar images? Show a diff!
1. Context of wider change.
   * Is there a ticket/issue describing the larger-scale work that this change is part of? Link to it! Render some text from it!
   * Are there follow-up changes? Does it build on previous changes?
   * Does this change depend on other pending changes? Should they be merged atomically?
      * Ideally changes should be very small, but navigable with the right context to understand the bigger picture. Being able to change that level of granularity ("show me the change to this file in this pull request", "show me the whole change of the whole stack of dependent pull requests") would be nice, too.
1. Previews.
   * Editing a markdown document? Render a preview, and render a diff!
   * Editing a web frontend? Host a preview of the site somewhere, and link to it.
1. An elegant, at a glance view of what I need to do, both per-review, and across all reviews.
1. Reviewers should be able to give code-edit suggestions.
   * Double-clicking on the code, fixing the typo, and presenting the code author with an "Accept suggestion" button is so much lower friction than saying "nit: It's just a typo, but there's an extra s here", and then requiring the code author to change branch, make an edit, commit, push, and say they've fixed it.
1. Automated reviewers should be easy to plug in.
   * If I can write a tool which detects problems, I can save humans time.
   * Doubly so, if they can propose code-edit suggestions which can be accepted at a button-press.
   * There are a few classes of these; "always correct" suggestions vs "maybe a good idea" suggestions.
1. Merging code by pressing a button in a web UI is nice.
   * Having an option to merge automatically when reviewers are happy is even nicer.

**What don't I want** in my code review tool?
1. Anything to do with the specifics of a version control system.
   * Maybe the system ingests things via git push or ingesting a patch file, and maybe it can merge things onto a branch, but those should be the only times version control matters.
1. Visual clutter.
   * Everything I want to know about a review should be quick and easy to discern at a glance. I should be able to dig in deeper when I need/want to.
1. An overly simplified, or overly strict, model of review.
   * Different projects, teams, companies, and people, use different models for code review requirements, and code ownership. Code review tools all seem to either treat all reviewers as equal, or assume that each reviewer ticks a certain box (generally "owns directory X"). If what I want to say is "I'm happy with this code, but Sophie should check it over", or "I'm happy with this code, but here are some trivial things you should address first", or "Two people should review every change", I want that to be obvious.

There are also some **crazy ideas** that I'd love to see, but which sound hard:

1. AST-aware formatting, diffing, and history.
   * It's crazy that people need to have style guides, and style wars, over when lines should wrap, or how much things should be indented. "Lines" of code is a weird concept. Statements and expressions are the building blocks of code, not lines!
1. Semantically aware change decomposition.
   * You renamed a field, then extracted a function, then added a new call to the function. Wouldn't it be cool if your code review tool could tell your reviewer that's what you did, rather than them inferring it from reading?
