If an application has a Service-oriented architecture and is designed to be Event-Driven in its' flows, what does that mean exactly?

Mainly that means that we want:

- a loosely coupled set of services (which are simpler to maintain because they only handle their own set of business capabilities, ideally, independent from others)
- finer-grained scalability (ability to add more consumers to specific capability)
- flexible extension points in our flows - so we can add some behaviour without rewriting every single place to support it
- an ability to deploy only some of the services (and not touch something that hasn’t changed)

That also means we can consider different options for communicating within our system. We can:

- make plain simple REST requests
- schedule an internal job for asynchronous execution
- post a message to shared queue so consumers can pull and process it (handled via SQS)
- publish a message directly to specific subscribers (handled via SNS)

Usually, we can distinguish between two forms of message between services:

- Command: we want to make other service to do something
- Event: we want to notify other services about something that happened (so they can react, but we do not expect that)

## Right methods for the right flows

Ultimately, there are a lot of options to choose from and our business flows can be composed from all of these methods at the same time. But how to pick the right method for a specific action?

### Synchronous Command

If you need to ensure that something specific happens: 
- now,
- you are interested in the results of that action. Or you need to get some data right now from elsewhere.

You may just make a REST request to another service. Use the following methods: 
- GET if you want to grab some data and there are no side-effects at all 
- PUT if you want to update server state and your change is guaranteed to be idempotent (calling it with the same data several times has the same effect as the first change) 
- POST if you want to change server state (e.g. DB) and your change might create new data (or side effects) every time you call it 
- DELETE if you want to remove certain entity from your server state 

When making REST requests between different BE services, consider: 
- retrying on temporarily erroneous responses, and whether it is OK for retrying not to be durable (e. g. you will not keep retrying after your service gets restarted) 
- amount of data you pass between services, as it will be lost after response is given. 

Good examples are: 
- And kind of FE - BE interaction 
- POST to start the translation job (even if we don’t see the results straight after call, we know the job has been scheduled by the response) 
- GET to retrieve segments from storage

### Asynchronous Command

If you need to ensure that something specific happens, but: 
- it may happen a bit later, 
- you are not interested in the results of that action right now
- but you would like to eventually get the results or retry if action has failed

You can publish a targeted message (e.g. SNS) so the specific consumers may handle it for you. Alternatively, you may choose to create an REST endpoint that persists message and enqueues the action. Such endpoints can request a **callback URL** and often return 202 Accepted to indicate that the effect will happen later.

Such message is still a Command, because:
- It is targeted 
- It still expects a certain side-effect later (and may wait for results and retry if they didn’t arrive)

Examples:
- Todo

### Asynchronous Event

If you simply need to notify others that certain fact has happened (e.g. translation was finished), and: 
- you are not interested in the side-effects directly
- you don’t care if anything happens at all

You can publish a message to the Queue (e.g. SQS) and other consumers can process that message independently from your flow. That message should be in the Event format, because we are representing what has happened and don’t expect a specific reaction. A distinctive trait is that events are named with verbs in past tense (e.g. `import.uploaded`). 

We don’t have good examples now, but potentially we could have: 
- An audit log service reacting to `translation.changed` event and writing a record to DB about that (storing the user who did that and other relevant data). A service that originated the event doesn’t control or even know about the side-effect - thus, they are decoupled.
- File conversion picking on `export.encoded` event and creating a ZIP bundle for export.

### Inter-service Delayed Command (a Background Job)

If you need to handle a long-running flow within your service, or have a particular action that you need to eventually succeed. Also, you can apply the same pattern to periodic jobs that happen regularly.

Just schedule that to the job processor and return early on the response (optimally - with some reference to that job so another side can query the status of such job afterwards). 

Examples: 
- email sending
- data cleanup
- generating some reports