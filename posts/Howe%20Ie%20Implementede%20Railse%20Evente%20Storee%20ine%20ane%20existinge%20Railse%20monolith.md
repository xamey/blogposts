If you don't know about Rails Event Store gem, you can check it out [here](https://railseventstore.org/). It's built by Arkency, and it's a gem that allows you to implement logic around Event Sourcing pattern in your Rails application. Very cool and open-source.

## Our current architecture

In my day-to-day job at Sêmeia, our monolith has grown over the years as we expanded our product offering, and adding one feature means adding either new models or existing models evolving.
We currently use parts of the Event Sourcing pattern, as most of our records in database are what we may call "events": when an user does a CRUD operation through our UI, an event is created. It may be a new model, which is in fact the very first event for a specific user, or it can be what we could call an "update" event: an event is created with the latest updated, while we consider the previous one soft-deleted.

The tricky part is that we may react to event creations/updates: imagine sending an email to an user right after its creation. The easy way is to use [ActiveRecord callbacks](https://guides.rubyonrails.org/active_record_callbacks.html) such as `after_create` or `after_update`. An example below:

```ruby
class User < ApplicationRecord
  def after_create(user)
    Mailer.with(chat_message: "Welcome to Sêmeia, #{user.name}!", user: user).deliver_later
  end
end
```

It works... but I don't find it satisfying for a few reasons:

- It's not the responsibility of the model to react to its creation: a model should only be aware of its attributes and its relationships. Callbacks should be used to compute derived attributes, not to react to events. In that way, the model shouldn't be aware of an action which is not related to its attributes and that should happen after its creation.
- Multiple business logic are highly coupled: an unit related to a "User" feature shouldn't be aware of a "Mailer" feature, it lacks of separation of concerns.
- We may use concerns to decouple model from side-effects, but in the end, the model still holds the business logic: what if we want to react to an event in a different way considering the context? We must implement either different concerns, or complexify implementation that end up being in the model, while we could use _adapters_. Wack!

![problem-architecture-coupling](https://github.com/xamey/blogposts/blob/main/imgs/problem-architecture-coupling.png?raw=true)

## Rails Event Store

We need that "something something" to decouple the models from the side-effects they may induce, and here comes Rails Event Store.
I won't go into details about the gem, but I'll show you how I implemented it in our monolith.

Rails Event Store introduces a pub/sub system, where you can subscribe to events and react to them through "handlers", aiming to remove the need to rely on ActiveRecord callbacks.

In our previous example, we could:

- Create a `UserCreated` event that is triggered when a `User` record is created.
- Create a `SendEmailAfterUserCreationEventHandler` that would react to the `UserCreated` event and send an email to the users.

That way:

- The `User` model doesn't need to know about the `SendEmailAfterUserCreationEventHandler` and the need to send an email.
- The `SendEmailAfterUserCreationEventHandler` is responsible for sending an email, holds a minimal amount of business logic, and can be easily maintained, tested and reused.
- We can easily react to the `UserCreated` event in a different way considering the context, by creating different handlers.

And voilà!

In order to implemented this pattern, we needed to define a few things to automatically generate the events that will be caught by the handlers, and how the handlers are defined.

### Trigger the events

```ruby
module TriggersEvent
  extend ActiveSupport::Concern

  included do
    after_create_commit :publish_create_event
    after_update_commit :publish_update_event
    after_destroy :publish_destroy_event

    private

    def publish_create_event
      publish_event("Created")
    end

    def publish_update_event
      publish_event("Updated")
    end

    def publish_destroy_event
      publish_event("Destroyed")
    end

    def event_store
      Rails.configuration.event_store
    end

    def publish_event(verb)
      return if @disable_dispatch_event

      event_class_name = "#{self.class.name}#{verb}"
      if Object.const_defined?(event_class_name)
        event = event_class_name.constantize.new(data: self)
        event_store.publish(event)
      end
    end
  end
end
```

`publish` method is Rails Event Store's method to publish an event on the bus, and waits an instance of a specific class that we will define later.

If I want any of my model to trigger such events, I just need to include the `TriggersEvent` concern in it. That's the "pub" part. Pretty simple once defined.

For example, if we include this concern in a `User` model, an `UserCreated` event will be published when a `User` record is created, and the `data` attribute will be the `User` instance.

```ruby
class User < ApplicationRecord
  include TriggersEvent
end
```

### Define the handlers

Then, let's talk about the "sub" part.

We will define classes that will react to the events we will publish.

```ruby
class SendEmailAfterUserCreationEventHandler
  def call(event)
    Mailer.with(chat_message: "Welcome to Sêmeia, #{event.data.name}!", user: event.data).deliver_later
  end
end
```

And that's it. It's just a regular class that implements a `call` method, which accepts a `RailsEventStore::Event` instance as an argument. `call` is executed when the handler reacts to the event.

For more information about he RailsEventStore::Event class, you can check [the source code](https://github.com/RailsEventStore/rails_event_store/blob/d2e7d1e3c1af12467cd83931aaf80514aef3ef76/ruby_event_store/lib/ruby_event_store/event.rb), as it is underneath a `RubyEventStore::Event` class.

A handler can also be asynchronous, I let you check [the documentation](https://railseventstore.org/docs/v2/subscribe/#async-handlers) for more information.

### Defining the publishers-subscribers mapping

We defined an initializer that will do the following:

- Instanciate the Rails Event Store that will be used among the application.
- Define the publishers-subscribers mapping in an intelligible way.
- Define the event classes that will be used by our publishers (the one that is passed as an argument to the `publish` method).

```ruby
# config/initializers/rails_event_store.rb
require "rails_event_store"

Rails.configuration.to_prepare do
  event_store = RailsEventStore::Client.new
  # event_store is a singleton that is available everywhere in the application
  Rails.configuration.event_store = event_store

  # Define the handlers and the events they will react to
  handlers = {
    SendEmailAfterUserCreationEventHandler => %w[
      UserCreated
    ]
  }

  handlers.each do |handler, events|
    events.each do |event|
      # For each handler x event couple, we :
      # - Define the event class if it doesn't exist. This will be our "UserCreated" class that inherits from RailsEventStore::Event and
      # which will be used in the `publish` method.
      # - Subscribe the handler to the event
      new_class = Class.new(RailsEventStore::Event)
      Object.const_set(event, new_class) if !Object.const_defined?(event)
    end

    event_store.subscribe(handler, to: events)
  end
end
```

Once defined, the only thing you will need to maintain is the mapping between the events and the handlers.
For example, if I want to notify an admin when a new user is created, I just need to add the new `NotifyAdminAfterUserCreationEventHandler` handler to the mapping.

```ruby
handlers = {
  NotifyAdminAfterUserCreationEventHandler => %w[
    UserCreated
  ]
}
```

And if I want a handler to react to different events, I just need to do the following:

```ruby
handlers = {
  NotifyAdminAfterUserOperationEventHandler => %w[
    UserCreated
    UserUpdated
  ]
}
```

All your developers have to do is to include the `TriggersEvent` concern in their models, and define the handlers they want to use.

![separation-of-concerns](https://github.com/xamey/blogposts/blob/main/imgs/separation-of-concerns.png?raw=true)

## Conclusion

I hope this article helped you understand how to implement Rails Event Store in your Rails application and invited you to decouple your models from side-effects. Our integration has been quite successful, anytime we groom a feature that requires a side-effect after a record creation/update, all we say is "we just need a handler".

If you have any questions, feel free to contact me on my socials.
