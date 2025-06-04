As a software developer, or more precisely as a web developer, we're here to address multiple challenges to please our users. Our web apps must maintain across years of development, address any edge cases that business may introduce, should be performant on both server and client side, deliver a user experience that feels instantaneous and enjoyable... and sometimes for millions of clients on a very limited timeline. Phew! It's what "full-stack" stands for, right?

In my pursuit of becoming a better web developer, I've spent a lot of time exploring architectural patterns that can help me meet these demands with clarity and efficiency.

What follows is a highly opinionated take on one such approach. It's shaped by my personal research and experiences (I'm not trained on the whole internet, after all). But I've found it valuable enough to share here.

## CQRS (Command Query Responsibility Segregation) and message-driven architecture

The backend architecture embraces Command Query Responsibility Segregation (CQRS) and asynchronous, message-driven processing.
The key concept here is: reads are fast, commands are complicated and asynchronous.
We won't worry about reads in this section as they will be addressed later, but we'll treat our back-end as a simple message broker, with all the authentication/authorization stuff.

When a client requests the back-end, it expresses a business intent that could modify the system (i.e., the data we persist in the database). For example, a client could try to "Delay a task from today to tomorrow": the request will be validated, and a message will be sent to the broker.

On the other hand, we have subscribers who listen to domain-specific commands and encapsulate business logic, who are responsible for performing modifications. One important thing to note is that those units should consider that they may accidentally receive the same message twice, ensuring idempotency.

This architecture has numerous benefits, and I'll pick the ones that I find interesting:

- almost everything is treated asynchronously, increasing resilience and scalability
- retry and dead-lettering add fault tolerance
- business logic is centralized, isolated, enhancing testability

There are a lot of existing tools or frameworks that allow us to implement such a pattern (any HTTP server framework with Kafka/RabbitMQ/Redis/... integration).

![cqrs-diagram](https://github.com/xamey/blogposts/blob/main/imgs/cqrs.png?raw=true)

## Optimistic updates

From the user's perspective, the application must feel fast and consistent-even when the backend processes commands asynchronously. We don't want our front-end to be full of painful loaders or skeletons.

Optimistic UI is a pattern where the front-end assumes that user commands will succeed and immediately reflects changes in the interface. For instance, if a user updates their profile picture, the UI should reflect the change instantly rather than waiting for the back-end response. In cases where the command fails, due to a validation error or conflict, the frontend reverts the changes, ensuring consistency.

![optimistic-update-diagram](https://github.com/xamey/blogposts/blob/main/imgs/optimistic.png?raw=true)

Perhaps, the front-end shouldn't mimic the whole business model which is encapsulated in the back-end: it is not its role, and it's counter-productive. It can optimistically update a subset of the modifications that may occur if the command succeeds. For instance, if updating a task command may modify another uncorrelated model, the UI shouldn't be aware of that and that uncorrelated model should be updated independently of the action.

## Local-first sync

By default, our front-end may render with:

- nothing (first load)
- cached data (previous session)
- optimistically updated data (recent user action).

But to remain consistent, it must frequently and automatically sync with the backend, fetching the most up-to-date state of the database. The database is our single source of truth, and our frontend should reflect it as closely as possible.

Combine this real-time sync with CQRS and optimistic UI, and you've created a virtuous cycle:

- the user issues a command.
- the UI immediately reflects the intent (optimistic update).
- the backend processes the command asynchronously.
- subscribers modify the database.
- the frontend receives the updated state in near real-time and reconciles with reality.

![circle](https://github.com/xamey/blogposts/blob/main/imgs/circle.png?raw=true)

Implementation of such architecture will be my next post, and will involve:

- [TanStack DB](https://github.com/TanStack/db)
- [electric-sql](https://electric-sql.com/)
- and a back-end with a message broker, maybe Rails
