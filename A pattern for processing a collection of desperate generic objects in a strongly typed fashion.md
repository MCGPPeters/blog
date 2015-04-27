A pattern for processing a collection of disperate generic objects in a strongly typed fashion
------------------------------------------------------------------------
When working on my open source project [Histrio](https://github.com/MCGPPeters/Histrio) (a set of libraries for building actor based systems), on multiple occasions I've had the need for processing objectsa collection of disperate generic objects as a single collection. At the same time I wanted to handle each individual object in a strongly type fashion when I was processing items from that collection. For example. An actor processes messages one at a time in the order the messages were received. This means messages have to be put into a single queue (the so called mailbox) per actor. This also means the queue needs to contain all the messages, no matter what type they are. 

A simple solution would be to put objects into the queue and cast them to a specific type. Be that as it may, in a system that potentially handles a huge amount of messages, this means a huge amount of type casting as well. In addition to that, an actor needs to respond to each message type in a different way (sending a message to an actor is somewhat of an equivalent to calling a method on an object). I would like some poor mans [pattern matching](http://en.wikipedia.org/wiki/Pattern_matching) so to say, based on the types of messages coming in.

For the sake of brevity, I will represent the handling of messages as a simple method like this:  
    
    public void ProcessMessage(IEnumerable<IMessage> messages)
    {
	    foreach (var message in messages)
	    {
	        message.GetHandledBy(_cellBehavior);
	    }
	}
    
The "messages" enumerable must be able to hold any kind of object. But remember, I want to process them in a strongly type fashion. For instance a "Get" and  a "Set" message, like these:

    public class Get
    {
        public Get(Address customer)
        {
            Customer = customer;
        }

	    public Address Customer { get; private set; }
    }
    
    public class Set<T>
    {
        public Set(T content)
        {
            Content = content;
        }

        public T Content { get; private set; }
    }
   
Because I can't put these disperate message types in one list, I must have a shared interface to do so. This is the IMessage interface shown below (ignore the method for now):

    internal interface IMessage
    {
        void GetHandledBy(Behavior behavior);
    }

So how can I put the `Get` and `Set` message into this same list. I don't want to alter the types of the `Get` and `Set`, but what I can do is define a generic type, `Message<T>`, that implements `IMessage` and will hold onto the `Get` and `Set` for me, like this (again, ignore the methods for now):

    public class Message<T> : IMessage
    {
        public Message(T body)
        {
            Body = body;
        }

        public T Body { get; private set; }

        public void GetHandledBy(Behavior behavior)
        {
            behavior.Accept(this);
        }

        internal void GetHandledBy(IHandle<T> behavior)
        {
            behavior.Accept(Body);
        }
    }

So adding messages to the `IEnumerable<IMessage> messages`  parameter, shown earlier, so that they can be handled by a `CellBehavior` may look something line this:

    messages.Add(new Message<Get>(new Get(..));
    messages.Add(new Message<Set<int>>(new Set<int>(..));

So now we have a way to add a set of disperate strongly type messages into a single collection / list.

On the other side of the equation, there should be component handling the messages, like `Get` and `Set`. In Histrio, a `Behavior` is responsible to handle certain types of messages. A `CellBehavior` is such a `Behavior` that can hold on to a value for me and send it to another component on request (the `Get` and `Set` messages) :

    public class CellBehavior<T> : Behavior, IHandle<Get>, IHandle<Set<T>>
     {
        private T _contents;

        public void Accept(Get message)
        {
            var reply = new Reply<T>(_contents);
            Actor.Send(reply, message.Customer);
        }

        public void Accept(Set<T> message)
        {
            _contents = message.Content;
        }
    }

The `CellBehavior` implements a certain interface, `IHandle<T>` twice (in this case), one time for each type of message it can react to. 

    public interface IHandle<in T>
    {
        void Accept(T message);
    }

Take a look again at the first lines of code in the post:

    public void ProcessMessage(IEnumerable<IMessage> messages)
    {
	    foreach (var message in messages)
	    {
	        message.GetHandledBy(_cellBehavior);
	    }
	}

You may notice that the `IMessage` asks the `CellBehavior` to handle it. But you also may have noticed there is an impedance mismatch between what the `CellBehavior` actually handles (namely strongly type instances of `Get` and `Set`) and `IMessage` that it is asked to handle. Here the `Behavior` base class comes in, from which the `CellBehavior` is derived. Also take a look again at the implementation of `Message<T>` and `IMessage`:

    public abstract class Behavior
    {
        public virtual void Accept<T>(Message<T> message)
        {
            var handler = this as IHandle<T>;
            if (handler != null)
            {
                message.GetHandledBy(handler);
            }
            else
            {
                throw new InvalidOperationException();
            }
        }
	}
	
	internal interface IMessage
    {
        void GetHandledBy(Behavior behavior);
    }
    
	public class Message<T> : IMessage
    {
        public Message(T body)
        {
            Body = body;
        }

        public T Body { get; private set; }

        public void GetHandledBy(Behavior behavior)
        {
            behavior.Accept(this);
        }

        internal void GetHandledBy(IHandle<T> behavior)
        {
            behavior.Accept(Body);
        }
    }

Lets imagine that `IEnumerable<IMessage>` contains a `Message<Get>` message and `ProcessMessages` is called. This leads to the sequence of steps that does the actually magic:

 1. On the `Message<Get>` a call is made to a method that accepts a `Behavior` . `Message<T>` has this method as a result of implementing  `IMessage`  => `"message.GetHandledBy(_cellBehavior)"`
 2. `Message<T>` then does a call back  to the `Accept<T>` method of the `Behavior`, **to handle it in its strongly typed form** . => `"cellBehavior.Accept<Get>(Message<Get> message)"`
 2. It then repeats the trick by checking if the `Behavior` implements the `IHandle<Get>` interface (thus `IHandle<T>` for that specific type of `T`) . If so it **calls back again to the `Message<Get>`** passing itself as the `IHandle<Get>` => `"message.GetHandledBy(_cellBehavior)"`
 3. Then for the last time the `Message<T>` calls back again to the `Behavior` (or more specifically, the `IHandle<Get>`) passing the body of the message (`Get`) as an argument **in a strongly typed fashion**


