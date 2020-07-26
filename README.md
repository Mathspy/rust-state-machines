# Notes on a future Rust state machine library

## Preface
State machines are great. One of the earliest models of computation. They remain to this day as one of the most solid ways of modeling state which itself remains as one of the most complex and contentious topics of computer science to this day.\
It's not hard to write state machines by mistake. They are too simple that they are being rediscovered every day by people who did not know about them and thus end up rediscovering them! And that's really cool. In fact let us rediscover them together right now!

Let's start by illustrating the simplest state machine, a toggle button (AKA a switch):

> INSERT A DIAGRAM OF A TOGGLE (ON, OFF) STATE MACHINE HERE

This is a [finite state machine] where there's two states: `ON` and `OFF`. A single type of input: `toggle`. The circles (nodes) represent the differing states and the lines between them (edges) represent the transitions between the states which are caused by inputs to the machine. The input's name is written over the line.

Now if you think about how would you represent this in code you'll probably think of a Boolean, and you wouldn't be wrong! A Boolean is definitely more than perfect in this situation and there's no need for anything more. But this is a simple state at the end of the day.

Next let's make a slider that has all the integers from 1 to 3 (1, 2, 3)

> INSERT A DIAGRAM OF 1, 2, 3 STATE MACHINE

It's rather interesting how each state can go to all other states, right? That's not always the case!

> INSERT A DIAGRAM OF TRAFFIC LIGHTS STATE MACHINE

Okay but let's go back to our slider example. We now want to expand our slider to support up to 10! How would we represent that? Will we truly add 7 more states just for this? Well what if we want to represent the state of ALL 32-bit integers? :sweat: That's QUITE a few states right there, friend!\
This is an inherit limitation to [finite state machine]s, they are not great at representing **infinite** state. As their name might imply! But instead of throwing all state machines in the trash can, how about we try to [_extend_](https://en.wikipedia.org/wiki/Extended_finite-state_machine) them?

The extension is rather trivial in this case, we will extend the finite state with infinite state cells that can each be filled with basically anything from simple integers to more complex strings or floats. Let's look at a fun example

> A DIAGRAM SHOWCASING A DAY OF SOMEONE'S LIFE WHERE IT STARTS WITH SLEEP THEN PROGRESSES TO BREAKFAST THEN WORK AND WORK WILL HAVE SOMETHING ATTACHED TO IT (WHAT THEY ARE WORKING ON TODAY) THEN THEY WILL GO BACK HOME AND RELAX/HAVE FUN WHICH WILL HAVE TWO ATTACHMENTS (WITH WHO AND WHAT ARE THEY DOING TOGETHER [with who enum includes alone])

Good work-life balance is key!

There's something to note about this example which is that even though it would make no sense for us to have `currently_working_on` if the finite state is `SLEEPING`, we can't really encode that and will have to just deal with having it as `null` or `None`. This is how the formal definition of extended state machines work. But what if we can do better? OH PSYCHE.

## Idea and vague implementation details

> Insert photo of Ferris waving here (the gif from Discord)

So if you have used Rust for a good portion of time you will likely instinctively be thinking of modelling the State as an enum (especially that we can attach types to our variants like `currently_working_on`) and the Inputs as an enum and you'd definitely not be wrong!\
You'd then make a function called `transition` that takes in a `State` and an `Input` and returns `State`, perfect eh? Yep! There's just one tiny tiny detail that this implementation sacrifices. Even though we know already that `(State::Sleep, Input::WakeUp) => State::Breakfast`, our function has to be ran for us to determine that. What if we can turn it into a mapping though? Something like a static `HashMap<(State, Input), State>`, right?

"Well, but what's the benefit? How is that any better than a simple match-only `transition` function?" I hear you saying from right there and the major benefit is static analysis and dynamic introspect! Let's start with the later. With a map it's extremely trivial to figure out "what's a valid input" or "what's a possible next state" without any duplications of our transition data! (Think how would you do this if you had a function? Would you have made a mapping especially for `State => nextPossibleStates`? And another map for possible inputs? Or would you have tried running the function with all possible inputs and figuring out if state changed then not valid input, and the collection of states we get from all those runs is all possible next states?)\
Let's dive into next benefit! Static analysis! Since we know all transitions statically, it's pretty trivial to analyse them or [even make visual analyses of them](https://xstate.js.org/viz/)

There's a catch (or two) though that you might have seen coming up. If our `State` or `Input` is an enum with types (i.e: `State::Working(currently_working_on)`) we won't really be able to store them at all (we could try to use the default of the types, if the type has a default, but that's such an arbitrary limitation). The second catch is that if we successfully gain the benefits we were promised, we lose some other benefits, namely `match`'s exhaustive checking! Can we find a way around both those catches?

To resolve the first catch we need a way to _strip_ the types off of the enums. I wish Rust had a built-in for us to do that. But it's not super important because with a quick macro we can create a new enum with `+Discriminate` in the name (`StateDiscriminate`), which doesn't have the types and has a simple `impl From<State> for StateDiscriminate`. If we do this on both the `State` and the `Input` we should fairly easily be able to have `HashMap<(StateDiscriminate, InputDiscriminate), StateDiscriminate>`. And since we are already deep in the macro ground, with another macro we can have the HashMap statically generated and available with pretty syntax.

Okay how about the second catch? We want to be able to know with confidence that our `transitions!` macro matched everything. I mean that's the macro's job to be honest but it can be complimented relatively nicely with this: https://docs.rs/enum-map/ for the final shape of:
```rs
EnumMap<StateDiscriminate, EnumMap<InputDiscriminate, Option<StateDiscriminate>>>
```
A map from each state to a map of each input to an optional state. The `Option` is because `(State::Working, Input::WakeUp)` should be `None`.

The last remaining piece of the puzzle is to remember this isn't "the end". The macro will not be able to guess how to use the types of `Input::GoTo(Friend)` to fill in output of `State::HavingFun(Option<Friend>, Activity)`. It could potentially generate a trait that has to be "filled in" or it could just contain itself some closures that do the actual mapping of `(State, Input) -> State`.

The goal here is to generate that mapping above statically at compile time WHILE still having some form of usage of discriminate unions (Rust Enums). That's a power not present in any other implementation of state machines I have seen (they usually follow the specification to the word and end up with infinite state completely independent of the finite states)

[finite state machine]: https://en.wikipedia.org/wiki/Finite-state_machine
