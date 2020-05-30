---
title: "The Codenames Theory of Naming"
date: 2020-05-19T20:30:00-07:00
editors:
categories:
  - programming
tags:
  - codenames
  - naming
  - theories
---

[Naming](https://martinfowler.com/bliki/TwoHardThings.html), [as they say](https://twitter.com/codinghorror/status/506010907021828096?lang=en), [is hard](https://xkcd.com/910/).

Bad naming has a real cost, and not just in terms of well-worn jokes. Names are the simplest interface in code, and a bad name is a bad interface: it leaves the reader uncertain about the behavior behind the name. If I can't tell what a function does from its name, I have to read the function's implementation to understand what's going on.

To borrow the framing John Ousterhout uses in [A Philosophy of Software Design](https://www.amazon.com/Philosophy-Software-Design-John-Ousterhout/dp/1732102201), bad names increase the complexity of the codebase, since I need to hold more stuff in my head at the same time in order to understand the behavior of the system. On a practical level, worse names mean more time writing code and more time reviewing code and more time fixing bugs from poorly-written and poorly-reviewed code.

So bad naming is bad. How can we make it easier to come up with good names?

# Codenames

[Codenames](https://en.wikipedia.org/wiki/Codenames_(board_game)) is a board game played with words. The game board consists of a 5x5 grid of words, some of which belong to the red team, some to the blue team, and some to neither team. Only one player on each team knows which words belong to which team, and that player has to give their teammates hints to get them to guess which words belong to their team. The core gameplay consists of the clue-giver providing a one-word hint to their team, then the team trying to figure out which words the clue-giver thinks are related to that hint.

From the clue-giver's perspective, the basic problem looks like this:

<figure>
    <img src="/assets/images/codenames_words.png" alt="alt text html" class="full">
</figure>

You have some words that you want your teammates to guess (the blue dots), and some words you *don't* want your teammates to guess (the red dots)[^other-cards]. You need to choose a hint that is strongly associated with your words and not strongly associated with the other team's words[^higher-dimensional].

Naming an object (variable/function/class/module) is essentially the same thing as giving a clue in Codenames.

You're trying to communicate to your teammates (coworkers, clients, open-source community members) that this object has some types of behavior (blue dots), and does not have other types (red dots) using a limited number of words[^java-camel]. For example, you might use your name to clarify how the object was produced (`PasswordCredentials`, `configFileJson`), how it will be consumed (`oauthRedirectUrl`, `AdminSettingsView`), over what duration it will be valid (`UserSession`, `TransactionLog`), whether it is or is not suitable for a particular purpose (`NonDeadlockingDatabaseClient`, `getJobStatusSync`, `MockClock`), etc.

# Corollary: know what your object is not

One thing the Codenames Theory of Naming makes explicit is that the first step to finding a good name for your object is to figure out the blue and red dots.

The blue dots are generally pretty straightforward: these are the things you want your readers to think about when they read your object's name. You probably have a good idea of what your object does and why the reader would care about it, so this shouldn't be too hard. If you're stuck here, explain to someone else why your object needs to exist. The nouns and verbs in that explanation are your blue dots.

The red dots can be substantially trickier: these are the things that you want your readers *not* to associate with your object. You want to distinguish your object from these things so that readers know the boundaries of what your object does. This requires a certain amount of imagination, because you have to put yourself in the shoes of your reader: if they stumble upon your object while trying to understand a piece of code, what assumptions might they have about your object's behavior? Which of those assumptions are wrong?

One way to populate the red dots for your object is to think of alternative objects that you could have used in the same place. What other approaches would be possible here, and what makes your object different from those other solutions? If you're building the second version of a system, the problems with the previous behavior are a good starting set of red dots[^no-new-foo].

If you want a list of reasonable alternatives, you need to have some idea of the context that the reader will have. This is generally what people mean when they say that naming is hard: it's hard to know what parts of the behavior need to be included in the name and which ones can be inferred from the context the reader brings with them. I don't have a silver bullet here, but in the absence of theory I suggest falling back to experimental results. Run your names by some other people and see what they think (for particularly important names, make sure you include a representative sample of the types of people who will be working with the object, especially people who aren't already familiar with that area of the code).

# Applying the theory

To apply the Codenames Theory of Naming next time you're coding,

1. Come up with a list of words that describe the behavior your object has or the way it is used
1. Come up with a list of words that describe behavior your object does NOT have or alternative approaches that readers might think your object would take
1. Find a small set of clue words that guide users to associate your object with the first list and distinguish your object from the second list
1. Build your name from the clue words, then validate the name with somebody else to make sure you've accounted for context

[^other-cards]: There are also other cards in the game, but those details don't matter to the analogy so I'm ignoring them.

[^higher-dimensional]: I've drawn the diagram in two dimensions, which obviously loses a lot of the nuance of the problem. The association between "buck" and "lion" is different from the association between "buck" and "check". It's more accurate to think of words as points in a high-dimensional space where each dimension corresponds to some aspect of meaning (the approach taken by [Word2vec](https://en.wikipedia.org/wiki/Word2vec)). In that framing, you're trying to establish a clean decision boundary between the blue dots and the red dots, and you have to communicate that decision boundary by providing another dot as a hint.

[^java-camel]: Some languages will conventionally allow longer names than others, but even in the more verbose languages, people get tired of discovering a new species of `MultipleAdjectiveAgglutinativeJavaCamel` on every line.

[^no-new-foo]: This should be obvious, but just tacking on `New` to the name is not a good way of distinguishing the new system from the old system. I sincerely hope that the difference between the two versions is more significant than "one was started after the other" - use that difference as a starting point. Also, if you call this version `New`, what will you call the next version? `Next`? [That would be silly.](https://en.wikipedia.org/wiki/Housing_at_the_Massachusetts_Institute_of_Technology#Undergraduate_dorms)
