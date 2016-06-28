Event Manager
=============

Event Dispatching allows developer to inject logic into an application easily.
Many frameworks implement some form of a event dispatching that allows users to
inject functionality with the need to extend classes.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119][].

[RFC 2119]: http://tools.ietf.org/html/rfc2119

## Goal

Having common interfaces for dispatching and handling events, allows developers
to create libraries that can interact with many frameworks in a common fashion.

Some examples:

* Security framework that will prevent saving/accessing data when a user
doesn't have permission.
* A Common full page caching system
* Logging package to track all actions taken within the application

## Terms

*   **Event** - An action that about to take place (or has taken place).  The
event name MUST only contain the characters `A-Z`, `a-z`, `0-9`, `_`, and '.'.
It is RECOMMENDED that words in event names be separated using '.'
ex. 'foo.bar.baz.bat'

*   **Listener** - A list of callbacks that are passed the EventInterface and
MAY return a result.  Listeners MAY be attached to the EventManager with a
priority.  Listeners MUST BE called based on priority.

## Components

There are 2 interfaces needed for managing events:

1. An event object which contains all the information about the event.
2. The event manager which holds all the listeners

### EventInterface

The EventInterface defines the methods needed for the EventManager to interact
with the listeners.

The event MUST contain a propagation flag that signals the EventManager to stop
passing along the event to other listeners. The event MUST contain an immutable
flag that signals to the listeners if they are allowed to modify the
content/payload of an event.

~~~php

namespace Psr\EventManager;

/**
 * Representation of an event
 */
interface EventInterface
{

    /**
     * Indicate whether or not contents/payload can be modified.
     *
     * @return bool
     */
     public function isImmutableEvent();

    /**
     * Has this event indicated event propagation should stop?
     *
     * @return bool
     */
    public function isPropagationStopped();

    /**
     * Indicate whether or not to stop propagating this event.
     *
     * @param  bool $flag
     */
    public function setPropagationStopped($flag = true);

}
~~~

### EventTrait

This is an example of an OPTIONAL compatible trait for a class that is
trying to implementing the EventInterface.

~~~php

namespace Psr\EventManager;

/**
 * Example trait for class implementing EventInterface.
 */
trait EventTrait {

    /**
     * Indicate whether or not contents/payload can be modified.
     *
     * @return bool
     */
     public function isImmutableEvent() {
        return $this->immutableEvent;
     }

    /**
     * Has this event indicated event propagation should stop?
     *
     * @return bool
     */
    public function isPropagationStopped() {
        return $this->propagationStopped;
    }

    /**
     * Indicate whether or not to stop propagating this event.
     *
     * @param  bool $flag
     */
    public function setPropagationStopped($flag = true) {
        $this->propagationStopped = (bool)$flag;
    }
    
    /**
     * @var bool $immutableEvent used to track if listener is allowed to change
     *                           the content of event outside of propagation.
     */
    private $immutableEvent = false;

    /**
     * @var bool $propagationStopped used to track if propagation should be
     *                               stopped.
     */
    private $propagationStopped = false;

}
~~~

### EventManagerInterface

The EventManager holds all the listeners for a particular event. For a mutable
event which can have many listeners that each return a result, the EventManager
MUST return the result from the last listener. For an immutable event the 
EventManager MUST return the result of EventInterface::isPropagationStopped().
The EventManager SHALL NOT attach the same listener multiple times for the
same eventName and priority but MUST ignore additional attach calls silently.
The EventManager SHALL allow the same listener to attach to multiple event and
priority combinations. The EventManager MUST allow multiple listeners for an
event with the same priority and MUST call them in the order they were attached
in. The EventManager MUST pass the parameters shown in the
EventListenerInterface when calling the attach listeners.

~~~php

namespace Psr\EventManager;

/**
 * Interface for EventManager
 */
interface EventManagerInterface
{
    /**
     * Attaches a listener to an event.
     *
     * @param string $eventName the name of the event to attach too
     * @param callable $listener a callable function
     * @param int $priority the priority at which the $listener executed with
     *                      higher priorities level listeners executing after
     *                      lower ones until isPropagationStopped() returns
     *                      true.
     *
     * @return $this fluent interface
     */
    public function attach($eventName, callable $listener, $priority = 0);

    /**
     * Detaches a listener from an event.
     *
     * @param string $eventName the name of the event to detach from
     * @param callable $listener a callable function
     *
     * @return bool true on success false on failure
     */
    public function detach($eventName, callable $listener);

    /**
     * Clear all listeners for a given event.
     *
     * @param  string $eventName the name of the event
     *
     * @return $this fluent interface
     */
    public function clearListeners($eventName);

    /**
     * Trigger an event
     *
     * Can accept an EventInterface or will create one if not passed.
     *
     * @param string         $eventName the name of the event
     * @param EventInterface $event
     * @param bool           $immutable this flag only applies if $event = null and Manager is creating a new instance.
     *
     * @return bool|EventInterface returns status of isPropagationStopped() for immutuble events or the possible changed
     * event for mutable events.
     */
    public function trigger($eventName, EventInterface $event = null, immutable = true);
}
~~~

### EventListenerInterface (non-nominal)

This is an OPTIONAL non-nominal example compatible interface for listeners used
to document the parameters that the EventManager will past to the listener when
the event is triggered. The listener SHALL NOT make any changes to the event
when isImmutableEvent() is true outside of that required by the implementation
of setPropagationStopped(). The listener MAY make other changes to the event
when isImmutableEvent() and they well be returned to the event trigger code if
the listener also uses EventInterface::setPropagationStopped(true).

~~~php

namespace Psr\EventManager;

/**
 * OPTIONAL non-nominal example interface for listener.
 */
interface EventListenerInferface {

    /**
     * Example callable listener method.
     *
     * @param string $eventName the name of the event
     * @param EventInterface $event the actual event object
     *
     * @return EventInterface returns an event with isPropagationStopped() set
     *                        if propagation should stop.
     */
     public function eventListener($eventName, EventInterface $event);

}
~~~
