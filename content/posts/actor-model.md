---
title: "Actor model"
date: 2025-05-25T12:19:58+05:30
description: "The actor model is a conceptual model to deal with concurrent computation. It defines some general rules for how the system's components should behave and interact with each other."
tags: [patterns, golang, actormodel, concurrency]
---

## What is an actor model?

[The actor model is a conceptual model to deal with concurrent computation. It
defines some general rules for how the system's components should behave and
interact with each other.](https://www.brianstorti.com/the-actor-model/)

In the Actor Model, an actor is the fundamental unit of computation. It's the
entity that receives messages and performs computations based on those messages.
But what exactly does "primitive unit of computation" mean? To draw a
comparison, in the threading model, a thread serves as the basic unit-having
multiple threads enables concurrency. Similarly, in the Actor Model, multiple
actors running concurrently allow for parallelism and distributed processing.

All the example from this code and the simple actor model implementation you
can find it in the github repository: [actor-model](https://github.com/rykth/actor-model)

### Actors are persistent

One of the key characteristics is that actors are persistent. This sets them
apart from threads, processes, or futures. For example, when you start a thread,
it performs its assigned task and then terminates once the work is complete. In
contrast, an actor doesn't simply disappear after handling a single message. It
remains alive and ready to process additional messages, maintaining its state
across multiple interactions. This persistence is fundamental to how actors
manage long-lived behavior and encapsulate state in concurrent systems.

### Internal state

In the Actor Model, actors encapsulate internal state, which is a defining
feature of this concurrency paradigm. Unlike futures, which don't really
maintain state, or threads that might have state through mechanisms like
thread-local storage, actors are designed to hold and manage their own internal
data. What makes actors unique is not just that they have state, but that they
encapsulate it-meaning their state is not directly accessible from the outside.
Instead, state can only be accessed or modified through message passing. This
encapsulation enforces a clear boundary around an actor's data, requiring
well-defined semantics for interaction, which helps avoid common concurrency
issues like race conditions and shared memory conflicts.

```go
type message struct {
	data string
}

type actorWithState struct {
	state int64 
	addr  actormodel.Address
}

const numberOfTries = 1_000_000

func (f *actorWithState) HandleMessage(msg any) {
	switch m := msg.(type) {
	case actormodel.Stopped:
		log.Println("Actor stopped")
	case actormodel.Started:
		log.Println("Actor started")
	case message:
		newState := atomic.AddInt64(&f.state, 1)
		if newState%1000 == 0 {
			log.Printf("Processed %d messages, current message: %s\n", newState, m.data)
		}
	}
}

func (f *actorWithState) Address() actormodel.Address {
	return f.addr
}

func main() {
	engine := actormodel.NewEngine()

	actor := &actorWithState{
		addr: actormodel.NewAddress("testing"),
	}

	if err := engine.RegisterActor(actor); err != nil {
		log.Fatalf("Failed to register actor: %v", err)
	}

	for i := 0; i < numberOfTries; i++ {
		if err := engine.SendMessage(
			actor.Address(),
			message{data: fmt.Sprintf("message #%d", i)},
			nil,
		); err != nil {
			log.Printf("Failed to send message #%d: %v", i, err)
			continue
		}
	}

	time.Sleep(2 * time.Second)
}

```

### Actors can create new actors

One of the actors fundamental capabilities is the ability to create new actors.
This is similar to how, in a threaded application, it wouldn't make sense if the
main thread couldn't spawn additional threads. The same idea applies here-actors
need to be able to create other actors to delegate tasks, manage complexity, and
build scalable systems. This dynamic creation of actors is a core aspect of the
model, enabling flexible and modular designs in concurrent applications.

```go
type parentActor struct {
	addr     actormodel.Address
	engine   *actormodel.Engine
	children children
}

func (p *parentActor) HandleMessage(msg any) {
	switch m := msg.(type) {
	case createChildMsg:
		childAddr := actormodel.NewAddress(m.childName)
		p.children.set(m.childName, childAddr)
		if err := p.engine.RegisterActor(&childActor{addr: childAddr}); err != nil {
			log.Printf("Failed to register child actor: %v", err)
			return
		}
	case sendToChildMsg:
		child, ok := p.children.get(m.childName)
		if !ok {
			log.Printf("Child %s does not exist", m.childName)
			return
		}
		if err := p.engine.SendMessage(child, message{m.msg}, p.addr); err != nil {
			log.Printf("Failed to send message to child: %v", err)
			return
		}
	case actormodel.Started:
		log.Printf("Parent actor started")
	case actormodel.Stopped:
		log.Printf("Parent actor stopped")
	}
}


type childActor struct {
	addr actormodel.Address
}

func (c *childActor) HandleMessage(msg any) {
	switch m := msg.(type) {
	case message:
		fmt.Printf("Received message in child: %s\n", m.data)
	case actormodel.Started:
		log.Printf("Child actor %s started", c.addr.ID())
	case actormodel.Stopped:
		log.Printf("Child actor %s stopped", c.addr.ID())
	}
}

func (c *childActor) Address() actormodel.Address {
	return c.addr
}
```

### Messaging

Actors can send messages to other actors and may respond to the sender zero or
more times. For example, when an actor receives a message, it might not respond
at all, which is perfectly valid. Alternatively, it could respond with a single
message-as we might typically expect-or with multiple messages, even an infinite
stream. This flexibility is a key aspect of how actors communicate.
Additionally, actors process exactly one message at a time. When multiple
messages are sent to an actor, they are placed in a mailbox-a queue-and the
actor processes them one by one. This sequential processing means that, within a
single actor, there is no internal concurrency. The Actor Model ensures that
each actor handles messages in isolation, maintaining consistency and avoiding
race conditions without the need for locks or shared state.

```go
type actorA struct {
	addr   actormodel.Address
	engine *actormodel.Engine
}

func (a *actorA) HandleMessage(msg any) {
	switch m := msg.(type) {
	case startCommunicationMessage:
		log.Println("Received start communication in Actor A")
		if err := a.engine.SendMessage(m.sendTo, firstMessage{data: "first message", sendTo: a.addr}, a.addr); err != nil {
			log.Printf("Failed to send first message: %v", err)
		}
	case secondMessage:
		log.Printf("Received second message in Actor A with: %s from: %s\n", m.data, m.sender.ID())
	case actormodel.Started:
		log.Printf("Actor A started")
	case actormodel.Stopped:
		log.Printf("Actor A stopped")
	}
}

type actorB struct {
	addr   actormodel.Address
	engine *actormodel.Engine
}

func (a *actorB) HandleMessage(msg any) {
	switch m := msg.(type) {
	case firstMessage:
		log.Println("Received first message in actor B")
		if err := a.engine.SendMessage(m.sendTo, secondMessage{data: "answer from the first message", sender: a.addr}, a.addr); err != nil {
			log.Printf("Failed to send second message: %v", err)
		}
	case actormodel.Started:
		log.Printf("Actor B started")
	case actormodel.Stopped:
		log.Printf("Actor B stopped")
	}
}
```

## Communication

Actors don't rely on channels like Golang. In Go, for instance, channels are
explicitly shared between goroutines and come with built-in semantics-like
whether they're bidirectional or unidirectional, typed or untyped, buffered or
unbuffered, and so on. Actors, by contrast, use a much simpler communication
model. There are no channels, no guaranteed delivery, and no buffering
semantics. Instead, messages are sent directly between actors using what's known
as best-effort delivery. This means the actor system trusts the underlying
protocol-whether it's an internal in-process mechanism or a network protocol
like TCP, UDP, or a Unix socket. If that protocol reports the message as sent
successfully, the actor assumes it was delivered. There's no built-in retry,
timeout, or acknowledgment mechanism. As a result, the Actor Model provides at
most once delivery: a message might be delivered once, or not at all, but never
more than once unless explicitly handled at a higher level.

## Supervision and error handling

How do we know when something goes wrong with an actor? How can we tell if
there's a real failure and not just a delay or a slow message? The answer lies
in a concept called supervision. In the Actor Model, supervision refers to the
practice of one actor monitoring and managing the state of another. This
supervisor actor keeps an eye on its child actors and can detect when something
goes wrong-whether it's a crash, a timeout, or unexpected behavior. Rather than
relying on timeouts or low-level error handling, supervision allows systems to
be built with self-healing properties, where supervisors can restart, stop, or
escalate issues when necessary. It's a powerful pattern for building resilient,
fault-tolerant systems.

So what exactly is supervision in the Actor Model? As mentioned earlier,
supervision is the ongoing monitoring of an actor's running state-whether it's
actively processing messages, has stalled, or encountered an unhandled
exception. When an actor runs into an issue it can't resolve on its own, it
turns to its supervisor for guidance. The supervisor, typically just another
actor, can then decide how to respond. It might choose to restart the failed
actor, hoping a fresh start will resolve the issue, or take other corrective
actions depending on the situation. This model creates a structured way to
handle failures gracefully and keep systems running reliably.

But then a natural question arises-who supervises the supervisor? This is where
the idea of supervision trees comes in. Supervision is organized hierarchically,
much like an organizational chart in a company. You have managers who oversee
individual employees, and those managers are themselves managed by higher-level
managers, and so on. Each supervisor knows how to manage its direct reports, and
is in turn managed by someone else. At the very top of the tree, there's usually
a root supervisor-something provided by the actor framework itself. This
top-level entity typically has built-in, default behavior for dealing with
exceptions or unhandled errors, and it rarely fails. This layered supervision
strategy is key to building resilient and fault-tolerant systems in the Actor
Model.

### Resources

* ["The actor model in 10 minutes"](https://www.brianstorti.com/the-actor-model/)    
* ["Hewitt, Meijer and Szyperski: The Actor Model (everything you wanted to know...)"](https://www.youtube.com/watch?v=7erJ1DV_Tlo)    
* ["Introduction to the Actor Model for Concurrent Computation"](https://www.youtube.com/watch?v=lPTqcecwkJg)