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
    <img src="/assets/images/codenames_words.svg" alt="alt text html" class="full">
</figure>

You have some words that you want your teammates to guess (the blue dots), and some words you *don't* want your teammates to guess (the red dots)[^other-cards]. You need to choose a hint that is strongly associated with your words and not strongly associated with the other team's words[^higher-dimensional].

Naming an object (variable/function/class/module) is essentially the same thing as giving a clue in Codenames. You're trying to communicate to your teammates (coworkers, clients, open-source community members) that this object has some types of behavior (blue dots), and does not have other types (red dots) using a limited number of words[^java-camel].

# An example

Suppose you're writing up a function that can only be called if a user has logged in and has a valid authentication token. Your function takes the user token and some other data, checks that the token is valid, and, if it is, does something with the data. What are you going to call this token?

## Blue dots

Let's start with our blue dots. What are some important concepts you want to convey about this token?

1. **It's a token** The word "token" already has a lot of meaning that you want to associate with this object. Also, "token" probably appears in our docs, so it's good to have that word appear in the code as well.
1. **It's associated with a user** This token proves that the user actually logged in, and it probably can be used to look up the user.
1. **It's used for authentication** This is actually just restating the previous point - the token proves that the user has logged in and can be used to get the user's identity. The token is also being used for authorization in this case, since we only allow this function to be called by users who have logged in.

At this point, `token`, `userToken`, and `user_auth_token`[^authn-or-authz] seem like reasonable names.

## Red dots

Now let's take a look at some potential red dots. What are things that you *don't* want readers to think about when they're reading this object's name?

1. **Tokens used for other purposes** Maybe auth tokens aren't the only types of tokens in your system. If someone reading this piece of code might reasonably mistake this object for one of those other types of tokens, you should include enough information in this object's name to make it unambiguous. This is particularly true if this function takes other types of tokens as inputs.
1. **Other types of tokens related to auth** A common pattern is to give users long-lived refresh tokens when they log in. These tokens can't be used to run commands themselves, but they can be redeemed for short-lived access tokens that can. Both types of tokens are auth-related, and many pieces of code will have to deal with both, but their use cases are different enough that it's worth being explicit about which is which.
1. **Auth tokens with different capabilities** Some tokens are self-describing bearer tokens: the token includes a list of actions, and anyone who has the token gets to perform those actions. Other tokens are opaque and need to be checked against some other source of truth to see if they're valid or get the list of actions they allow. If your system allows both, you may want to specify which one you're dealing with[^stringly-typed].
1. **Auth tokens with different levels of trust** If you haven't verified that your token is legitimate (either by checking its signature or asking an external source of truth), you shouldn't use that token to perform any actions. This is a big deal from a security perspective so it's a good idea to explicitly name untrusted tokens as such[^stringly-typed-2].

So now you have some additional constraints that might affect how you name your object. If you need to differentiate your token from tokens used for other purposes, you probably need to include `auth` or something like it in the name - `token` will be too ambiguous. If you have multiple flavors of auth token floating around in the same functions, you can't just use `authToken` (try `access_token` or `refreshToken` instead). If there are other important aspects of how your token must be used, you should include those (`bearerToken`, `unverified_auth_token`, etc.)

## Choosing a name

Now that you have your constraints, let's choose a name. One approach is to shove all of the information you can into the name: `verifiedUserAuthBearerToken`. Ick.

This name works, but it's huge and unwieldy. It's possible that your readers will need all that information (in which case you may want to think about refactoring your code to reduce some of the possible confusion).

Most likely, some of that information is already obvious from the context. Note that all of the potential red dots depend on what other types of objects exist in your system. There's no need to differentiate your object from other objects that a user would already know not to expect in your codebase. You can probably remove some of those extra adjectives and get a cleaner name.

# Corollary: know what your object is not

One thing this approach makes explicit is that the first step to finding a good name for your object is to figure out the blue and red dots.

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

[^authn-or-authz]: Since this token is used for authentication and authorization, it's probably fine to be imprecise here.

[^stringly-typed]: If you know that you'll only ever have one type of token at this point in the code, it's probably better to capture that information in the type system rather than the name (assuming your language has a type system that is expressive enough to allow that). That way the compiler can verify your assumption that you'll only ever get one type of token at that point and protect you from future refactorings that might allow other types of tokens to sneak into that piece of code.

[^stringly-typed-2]: As in the previous footnote, it's better to use your language's type system to differentiate between a trusted and an untrusted token, again assuming that it's powerful enough to do so.

[^no-new-foo]: This should be obvious, but just tacking on `New` to the name is not a good way of distinguishing the new system from the old system. I sincerely hope that the difference between the two versions is more significant than "one was started after the other" - use that difference as a starting point.
