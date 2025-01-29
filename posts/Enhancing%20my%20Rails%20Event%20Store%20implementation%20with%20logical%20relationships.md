In my first article, [I explained how I implemented Rails Event Store in an existing Rails monolith](https://xamey.xyz/post/?sha=2c00fc79252ddb1fc8c6788ff72752daa8d91127&title=How%20I%20implemented%20Rails%20Event%20Store%20in%20an%20existing%20Rails%20monolith) in a simple and automated way.
Perhaps, it was missing logical relationship between ActiveRecord models that were created through events handling.

## What has been added

To give a quick overview, I've added callbacks that can be executed from any ActiveRecord model that includes the right concern.
When the ActiveRecord is created or updated, an event is triggered, either about the creation or the update of the ActiveRecord.

![rails-event-store-current-architecture](https://github.com/xamey/blogposts/blob/main/imgs/rails-event-store-current-architecture.png?raw=true)

This is encapsulated in a concern, so it holds all the logic in one place, easing maintenance and testing.

```ruby
class User < ApplicationRecord
  include TriggersEvent
end
```

But as you can see in the image above, there's no relationship between the ActiveRecords and the subsequent events that are created or updated, which I find missing for various reasons.

![res-relationship-problem](https://github.com/xamey/blogposts/blob/main/imgs/res-relationship-problem.png?raw=true)

## Causation and correlation

The motivation that led to this modification is to be able to ease the debugging process of our application. For instance, considering one ActiveRecord instance, I would like to know:

- which ActiveRecord led to my creation or update, if it's the case
- which ActiveRecords have been created or updated after a reaction to my own creation or update

So I'll be able to construct a causal graph of my application, and be able to understand the flow of events and the relationships between ActiveRecords.

Rails Event Store provides a method to link events between them, thanks to the event's API.

```ruby
event.correlate_with(other_event)
```

It's as simple as that. It will automatically add the right identifiers in the event's metadata, which are stored in your database.
See the image below, which comes from [this article](https://railseventstore.org/docs/core-concepts/correlation-causation) to understand how it works.

![causation-correlation-events.png](https://github.com/xamey/blogposts/blob/main/imgs/causation-correlation-events.png?raw=true)

## Provide an event to another in our architecture

As my events are automatically created in an ActiveRecord callback, I need to provide the previous event to that specific callback execution, so I'm able to link them together.

![rails-event-store-current-with-link.png](https://github.com/xamey/blogposts/blob/main/imgs/rails-event-store-current-with-link.png?raw=true)

A naive solution would be to pass the event as an argument to the ActiveRecord constructor so it's available in the callback, but that would break the single responsibility principle, and overhead the model with logic that is not related. Developers would have to know about the event, and may forget about it, breaking the logic one day or another.

I found another solution which both encapsulates the logic in the related concern (in our case, `TriggersEvent`), and provides the event to the callback, by using the [RequestStore gem](https://github.com/steveklabnik/request_store).
It enables a global storage of data that is local to a request, and is thread-safe.

I just store the event that a handler reacted to :

```ruby
# handler.rb
RequestStore.store[:event] = event
```

And then, in the callback, I can retrieve it :

```ruby
# triggers_event.rb
previous_event = RequestStore.store[:event]
if previous_event
  event.correlate_with(previous_event)
end
```

![rails-event-store-current-with-link-and-request-store.png](https://github.com/xamey/blogposts/blob/main/imgs/rails-event-store-current-with-link-and-request-store.png?raw=true)

And our events are automatically correlated with the previous event.

Cool stuff in our case is that we include the `TriggersEvent` concern in `ApplicationRecord`, so we _always_ trigger events from an ActiveRecord callback, leading to the construction of a causal graph of our application with just a few lines of code.

## Enhance our API to navigate through the causal graph

Ok, events are correlated with each other, that's cool. But I would like to navigate between ActiveRecords.

It required one more hack! I'll rely on streams to able to retrieve the events that have been triggered, from an ActiveRecord instance. You can read more about streams [here](https://railseventstore.org/docs/core-concepts/link).

When I publish my event, I add a stream name to it, which is the name of the ActiveRecord class, and its id.

```ruby
  def event_store
    Rails.configuration.event_store
  end
```

```ruby
event_store.publish(event, stream_name: "#{self.class.name}-#{self.id}")
```

Then, I can retrieve the last event that has been published for a given ActiveRecord class and id :

```ruby
def related_event
  event_store.read.stream("#{self.class.name}-#{self.id}").to_a.last
end
```

Then, I can retrieve the ActiveRecord that triggered my creation/update:

```ruby
def triggered_by_record
  causation_id = related_event.causation_id
  return nil if causation_id.blank?

  # event's data is the ActiveRecord instance
  event_store.read.event(causation_id)&.data
end
```

And finally, I can retrieve the events that have been triggered by my related event :

```ruby
def triggered_events
  event_store
    .read
    .stream("$by_causation_id_#{related_event.event_id}")
    .to_a
    .map(&:data)
end
```

That `$by_causation_id_#{related_event.event_id}` stream name is automatically created by Rails Event Store when you:

- correlate events with the `correlate_with` method
- add the right subscription in your Rails Event Store configuration :

```ruby
  event_store.subscribe_to_all_events(RailsEventStore::LinkByCorrelationId.new)
  event_store.subscribe_to_all_events(RailsEventStore::LinkByCausationId.new)
```

And that's it! I can now navigate through the causal graph of my application, and understand the relationships between ActiveRecords.

## Conclusion

Whole code is available below :

```ruby
module HandlerCommon
  extend ActiveSupport::Concern

  def call(event)
    RequestStore.store[:event] = event
    # do some more stuff
    run
  end
end
```

```ruby
# This is an example of a handler that reacts to an event
class AnyHandler
  include HandlerCommon

  def run
    # do something
  end
end
```

```ruby
module TriggersEvent
  extend ActiveSupport::Concern

  included do
    after_create_commit :publish_create_event
    after_update_commit :publish_update_event
    after_destroy :publish_destroy_event

    def related_event
      event_store.read.stream("#{self.class.name}-#{self.id}").to_a.last
    end

    def triggered_by_record
      causation_id = related_event.causation_id
      return nil if causation_id.blank?
      event_store.read.event(causation_id)&.data
    end

    def triggered_events
      event_store
        .read
        .stream("$by_causation_id_#{related_event.event_id}")
        .to_a
        .map(&:data)
    end

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
      event_class_name = "#{self.class.name}#{verb}"
      if Object.const_defined?(event_class_name)
        event = event_class_name.constantize.new(data: self)

        if RequestStore.store[:event]
          event.correlate_with(RequestStore.store[:event])
        end

        event_store.publish(
          event,
          stream_name: "#{self.class.name}-#{self.id}"
        )
      end
    end
  end
end
```

```ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  include TriggersEvent
end
```
