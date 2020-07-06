---
title: "Code Names"
date: 2020-07-06T11:52:00-07:00
editors:
  - Bianca Homberg
categories:
  - programming
tags:
  - codenames
  - naming
---

[Naming](https://twitter.com/codinghorror/status/506010907021828096?lang=en), [as they say](https://xkcd.com/910/), [is hard](https://martinfowler.com/bliki/TwoHardThings.html).

Bad naming has a real cost, and not just in terms of well-worn jokes. Names are the simplest interface in code, and a bad name is a bad interface: it leaves the reader uncertain about the behavior behind the name.

If I can't tell what a function does from its name, I have to read the function's implementation to understand what's going on, which means I need to hold more stuff in my head at the same time in order to understand the behavior of the system. This is bad because my working memory is bounded, and as the code asks me to keep more ideas in my mind at the same time, I get slower and less accurate at remembering them all. On a practical level, worse names mean more time writing code and more time reviewing code and more time fixing bugs from poorly-written and poorly-reviewed code.

So bad naming is bad. How can we make it easier to come up with good names?

# Codenames

[Codenames](https://en.wikipedia.org/wiki/Codenames_(board_game)) is a board game played with words. The game board consists of a 5x5 grid of words, some of which belong to the red team, some to the blue team, and some to neither team. Only one player on each team knows which words belong to which team, and that player has to give their teammates clues to get them to guess which words belong to their team. The core gameplay consists of the clue-giver providing a one-word clue to their team, then the team trying to figure out which words the clue-giver thinks are related to that clue.

If you're the clue-giver for the blue team, the basic problem looks like this[^higher-dimensional]:

<figure>
    <img src="/assets/images/codenames_words.svg" alt="alt text html" class="full">
</figure>

You have some words that you want your teammates to guess (the blue words), and some words you *don't* want your teammates to guess (the red words)[^other-cards]. The challenge comes from the fact that some of your words may be very similar to the other team's words (for example, "bill" and "check" can be synonyms, and "plot" can be associated with "play" or "bomb" depending on how your team interprets the word). You need to choose a hint that is strongly associated with your words and not strongly associated with the other team's words.

Naming an object (variable/function/class/module) is essentially the same thing as giving a clue in Codenames. You're trying to communicate to your teammates (coworkers, clients, open-source community members) that this object has some types of behavior (blue words), and does not have other types (red words) using a limited number of words[^java-camel].

# An example

To illustrate this analogy, let's use an example that's reasonably close to something I've worked on recently: user validation.

Let's say you're writing a new function for your application, and this function can only be called if a user has logged in and has a valid authentication token. Your function takes the user token and some other data, checks that the token is valid, and, if it is, does something with the data.

What are you going to call the token?

## Blue words

Let's start with our blue words. What are some important concepts you want to convey about this token?

1. **It's a token** The word "token" already has a lot of meaning that you want to associate with this object. Also, "token" probably appears in your docs, so it's good to have that word appear in the code as well.
1. **It's associated with a user** This token proves that the user actually logged in, and it probably can be used to look up other information about the user.
1. **It's used for authentication** This is just restating the previous point - the token proves that the user has logged in and can be used to get the user's identity. The token is also being used for authorization in this case, since we only allow this function to be called by users who have logged in.

At this point, `token`, `userToken`, and `user_auth_token`[^authn-or-authz] seem like reasonable names.

## Red words

Now let's take a look at some potential red words. What are things that you *don't* want readers to think about when they're reading this object's name?

1. **Tokens used for other purposes** Maybe auth tokens aren't the only types of tokens in your system. If someone reading this piece of code might reasonably mistake this object for some other type of token, you should include enough information in this object's name to make it unambiguous. This is especially true if this function takes other types of tokens as inputs[^stringly-typed].
1. **Other types of tokens related to auth** Many auth systems will use [refresh tokens](https://auth0.com/learn/refresh-tokens/) in addition to access tokens. Both types of tokens are auth-related, and many pieces of code will have to deal with both, but their use cases are different enough that it's worth being explicit about which is which.
1. **Auth tokens with different capabilities** Some access tokens are self-describing bearer tokens: the token includes a list of actions, and anyone who has the token gets to perform those actions. Other tokens are opaque and need to be checked against some other source of truth to see if they're valid or get the list of actions they allow. If your system allows both, you may want to specify which one you're dealing with.
1. **Auth tokens with different levels of trust** If you haven't verified that your token is legitimate (either by checking its signature or asking an external source of truth), you shouldn't use that token to perform any actions. This is a big deal from a security perspective so it's a good idea to explicitly name untrusted tokens.

So now you have some additional constraints that might affect how you name your object. If you need to differentiate your token from tokens used for other purposes, you probably need to include `auth` or something like it in the name - `token` will be too ambiguous. If you have multiple flavors of auth token floating around in the same functions, you can't just use `authToken` (try `access_token` or `refreshToken` instead). If there are other important aspects of how your token must be used, you should include those (`bearerToken`, `unverified_auth_token`, etc.)

## Choosing a name

Now that you have your constraints, let's choose a name. One approach is to shove all of the information you can into the name: `verifiedUserAuthBearerToken`.

Ick.

This name works, but it's huge and unwieldy. It's *possible* that your readers will need all that information, but in that case you may want to think about refactoring your code to reduce some of the possible confusion.

Most likely, some of that information is already obvious from the context. Remember that all of the potential red words depend on what other types of objects exist in your system: there's no need to differentiate your object from other objects that a user would already know not to expect. You can remove some of those extra adjectives and get a cleaner name.

# Corollary: know what your object is not

One thing this approach makes explicit is that the first step to finding a good name for your object is to figure out the blue and red words.

The blue words are generally pretty straightforward: these are the things you want your readers to think about when they read your object's name. You probably have a good idea of what your object does and why the reader would care about it, so this shouldn't be too hard. If you're stuck here, explain to someone else why your object needs to exist. The nouns and verbs in that explanation are your blue words.

The red words can be substantially trickier: these are the things that you want your readers *not* to associate with your object. You want to distinguish your object from these things so that readers know the boundaries of what your object does. This requires a certain amount of imagination, because you have to put yourself in the shoes of your reader: if they stumble upon your object while trying to understand a piece of code, what assumptions might they have about your object's behavior? Which of those assumptions are wrong?

One way to populate the red words for your object is to think of alternative objects that you could have used in the same place. What other approaches would be possible here, and what makes your object different from those other solutions? If you're building the second version of a system, the problems with the previous behavior are a good starting set of red words[^no-new-foo].

If you want a list of reasonable alternatives, you need to have some idea of the context that the reader will have. This is generally what people mean when they say that naming is hard: it's hard to know what parts of the behavior need to be included in the name and which ones can be inferred from the context the reader brings with them. I don't have a silver bullet here, but in the absence of theory I suggest falling back to experimental results. Run your names by some other people and see what they think (for particularly important names, make sure you include a representative sample of the types of people who will be working with the object, especially people who aren't already familiar with that area of the code).

# tl;dr

I've found that using a Codenames-inspired approach makes it easier to come up with good names for software objects:

1. Come up with a list of words that describe the behavior your object has or the way it is used ("blue words")
1. Come up with a list of words that describe behavior your object does NOT have or alternative approaches that readers might think your object would take ("red words")
1. Find a small set of clue words that guide users to associate your object with the blue words and distinguish your object from the red words
1. Build your name from the clue words, then validate the name with somebody else to make sure you've accounted for context

[^other-cards]: There are also other cards in the game, but those details don't matter to the analogy so I'm ignoring them.

[^higher-dimensional]: I've plotted the words in two dimensions according to how closely related I think they are, which obviously loses a lot of the nuance of the problem. The association between "buck" and "lion" is different from the association between "buck" and "check". It's more accurate to think of words as points in a high-dimensional space where each dimension corresponds to some aspect of meaning (the approach taken by [Word2vec](https://en.wikipedia.org/wiki/Word2vec)). In that framing, you're trying to establish a clean decision boundary between the blue words and the red words, and you have to communicate that decision boundary by providing another word as a clue.

[^java-camel]: Some languages will conventionally allow longer names than others, but even in the more verbose languages, people get tired of discovering a new species of `MultipleAdjectiveAgglutinativeJavaCamel` on every line.

[^authn-or-authz]: Since this token is used for authentication and authorization, it's probably fine to be imprecise here.

[^stringly-typed]: If you have different types of tokens, it's probably better to use your language's type system to say which token is which type rather than relying on names (assuming your language has a type system that is expressive enough to allow that). That way the compiler can verify your assumption that you'll only ever get one type of token at that point and protect you from future refactorings that might allow other types of tokens to sneak into that piece of code.

[^no-new-foo]: This should be obvious, but just tacking on `New` to the name is not a good way of distinguishing the new system from the old system. I sincerely hope that the difference between the two versions is more significant than "one was started after the other" - use that difference as a starting point.
