## A Finite State Machine (FSM) Implemented in Python via Coroutines

I've been reading about the `yield` keyword in Python lately, and in the process I learned about the *coroutine* design pattern. I read about it in JavaScript a while back and played a bit with generator functions. The languages are pretty similar as far as how generator functions work.

This fascinating rabbit hole involving both languages had me engaging with the following amazing resources — all of which I highly recommend: 

- http://www.dabeaz.com/coroutines
- https://www.geeksforgeeks.org/coroutine-in-python
- https://www.codementor.io/@arpitbhayani/building-finite-state-machines-with-python-coroutines-15nk03eh9l 
- https://x.st/javascript-coroutines/ 
- https://github.com/getify/You-Dont-Know-JS/tree/1st-ed/async%20%26%20performance 

So yeah. 

Before we get started, I'd appreciate it if you show your support by liking this article, sharing, and subscribing. Thanks so much!

## Why Finite State Machines?

You can do more than FSMs with coroutines, to be sure. I'd like to explore those other use cases in the future. But for now, since so many people (myself included) are so familiar with the Object-Oriented Programming ways of the world, an FSM seems like the perfect intro to the power that coroutines have to help you manage application state.

In this article, I'd like to use coroutines in Python to model a classic finite state machine example: a vending machine. The REPL is embedded of course so you can play with it. 

## Designing a Finite State Machine for a Vending Machine

Our example will be a Japanese style vending machines that sells umbrellas for $5 apiece. It only takes certain denominations of the U.S. Dollar.

It only sells two types of umbrellas (a lacy feminine one for walks in the park, and a plain, black one that is compact enough to fit in a businessman's suitcase) since the vendor doesn't have a lot of competitors in the american otaku market. 

Additionally, we will assume the vending machine never runs out of stock for economic reasons: otaku demand in this timeline will never outpace umbrella supply. Similarly, it never runs out of change. 

This example is a perfect computational problem to model with a finite state machine! Here are the qualities of a finite state machine: 

- a finite number of... states (lol)
- an finite set of input tokens, also called an input alphabet
- a transition function that maps the current state of the machine to the next state based on the input token; it should also produce relevant output at that time if there is any

It's common to see FSMs modeled in pretty yet IMO intimidating diagrams with circles and arrows. Instead, I prefer to model them in a table that you could easily make in a spreadsheet or word document. Hey, it's a lower budget solution.

¯\\\_(ツ)\_/¯

[Here's a link to the one I made for our vending machine.](https://docs.google.com/spreadsheets/d/1VlDIWJAdUApdQdB32etVGOCNcMmEaHomHfVdMbWib60/edit?usp=drivesdk)  

To avoid confusion, the states are named with letters (as opposed to numbers) in the first column of the diagram. The valid inputs to the vending machine are: a $1 bill, a $2 bill, a $5 bill, a press of the L-Button to request a Lacy Umbrella, and a press of the B-Button to request a Black Umbrella. 

Depending on which state we are currently in: 

- a given input may or may not change the state of the machine to a different state 
- a given input may or may not produce output in the form of currency or otaku umbrellas

The starting state of the machine is A, which is when there are no dollars held in the machine available to be used toward the purchase of an umbrella. State B is when there is exactly one dollar held in the machine available to be used toward the purchase of an umbrella. 

And so on and so forth until State F, where there are five dollars held in the machine, which is the required amount for the machine to be holding in order to produce an umbrella when one of the two umbrella buttons is pushed. 

When that happens, the machine returns to State A, and you can put in more money to buy another umbrella. Gotta collect 'em all! ☂️🌂

## What does a Class-Based Approach Look Like?

Before we implement this FSM with a coroutine, let's give the OOP approach a shot. Here is an implementation with a simple Python `class`:

%[https://repl.it/@dvanderweele/VendingMachineFSMviaClass]

As you can see, rather than rely on an arithmetic algorithm that involves adding and subtracting quantities of currency, I explicitly defined all the relationships between states and inputs and outputs. That's certainly not the only way to define such functions, but it is explicit in a way similar to what you might see in a Discrete Mathematics textbook.

In the event an unrecognized token is submitted to the Vending Machine, it is simply returned directly. If a surplus of currency is submitted, the difference is returned. 

Since this is a highly contrived example, there is no need to store the options for different states, inputs, and outputs anywhere other than within tuples in class attributes. However, for a real vending machine with exhaustible inventories of umbrellas and currency, you might explore other options.

## How do we build the coroutine version?

The ability to implement coroutines in Python and JavaScript is derived from the `yield` keyword, which is used in a very similar way in both languages to implement generator functions. 

A regular function — or if you're feeling a bit haughty, a *subroutine* — executes from start to completion whenever called, consuming and returning (or not) one or more values. It runs until a `return` statement is encountered.  The next time it is called, execution begins from the start.

A generator function — or *semicoroutine* for the haughtily inclined — is a callable sequence of instructions like the regular function. However, where it differs is that instead of `return` statements, a generator function will have statements that utilize the `yield` keyword. 

The `yield` keyword will indeed "return" a value to the caller. Where it's different is that, at that point, execution of the generator function is not *terminated* until the next time it is called — it's *paused*. 

It seems like a subtle difference, but it's actually rather powerful. 

Execution does not begin "anew" from the top every time a generator function is called. The first time a generator function is called, execution will:

- advance from the beginning of the function to the first `yield` expression
- return a generator function object to the caller 
- pause

That returned generator function object can then be iterated over in a loop or by directly calling the `next` method on it. Every time that happens, execution picks up where it left off (with any state local to the function preserved from last time) and keeps going until the `yield` keyword is encountered, this time returning a value based on the evaluated expression. 

In simple terms, what this means is you can put a `yield` statement inside a loop (oftentimes an infinite one), and every time the `next` method is called on the generator object it will advance the loop by one tick and return a value at the`yield` statement. 

The `yield` keyword does have another critical trick up its sleeve: *it'll accept a value too.*

This is possible in part because `yield` can be a part of an assignment expression. Whereas a *semicoroutine* formed with the `yield` keyword will keep spitting out a sequence of values to you, a *coroutine* is made by repeatedly sending in values. Instead of `next`, you can use the `send` method to send in a value to the `yield` keyword, and then execution will resume from that point. 

You can use the `yield` keyword in multiple places within the same routine to provide an alternating pattern of consumption and production behaviors. However, trust me, it'll have you pulling your hair out. The best option, like in my example, is to pass in a *sink* function to your coroutine. 

The sink is the final destination of the values produced by a coroutine or chain of coroutines. It could itself be a coroutine, although in this case because it is just printing values it's a simple lambda function.

Here's my coroutine implementation of our umbrella vending finite state machine:

%[https://repl.it/@dvanderweele/VendingMachineFSMviaCoroutine]

**Real talk:** it's cool, but not quite as intuitive as the class-based version for those of us who are more used to traditional OOP. 

## So why would anyone do it this way?

I don't know that you would, other than to get practice with the `yield` keyword. Or if total privacy of internal attributes/state matters a lot to you. 

In that way, it's an alternative to class-based syntax in a similar way that people use JavaScript's intuitive closure mechanism as an alternative to JavaScript's prototypal object orientation. 

The real power of coroutines isn't necessarily in their ability to implement state machines (which in itself is fun to practice), but in their ability to form data manipulation pipelines and even to implement concurrency. To that end, I can't recommend the resources mentioned at the beginning enough.

Thanks for reading! I'd love to hear from you below if you have any thoughts. Don't forget to like and subscribe. 

*Till next time, keep on coding* 😎🤙