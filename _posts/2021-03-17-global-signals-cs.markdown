---
title:  "Global signals in C#"
repository: global-signals
preview: /assets/images/posts/2021-03-17-global-signals-cs/signal-preview.gif
date:   2021-03-17 10:00:00 +0300
categories: cases
tags: [C#, unity3d]
lang: En
layout: post
---

This time I will share with you my implementation of the interaction between different parts of the application, through the transmission and processing of signals. This rather simple method solved for me the problem of transferring data from one object to another when using multiscenes in **Unity3d**.

## Prolog
As mentioned above, when creating an application in Unity3d, I came across the fact that it is necessary to transfer data from one object to another.
Moreover, this was complicated by the use of multitsen. My project involves creating a lot of scenes and I really did not want to manually implement dependencies.
Methods like `FindObjectsOfType` were also not suitable for me, as they take up too many resources to search for objects. I wanted to simplify data transfer between scene objects as much as possible without compromising performance.

As I delved deeper into the various architectural patterns, I came across **ECS** and decided that I had found a silver bullet that would kill all my monsters. After trying out the most popular frameworks in this area, I chose [Actors](https://github.com/PixeyeHQ/actors.unity). Its authors are very friendly guys, they even allowed me to make some changes regarding the loading of scenes from **AssetBundle**. 

But as is often the case with frameworks, it provided me with huge opportunities, most of which I did not intend to use inside my project. Basically I used a structure called **Signals**. The author himself described it something like this: _"it sends a message literally into the void, and if there is a recipient for it, it will receive it in any part of the application"_. This more than covered the entire range of my tasks to ensure interaction between objects in scenes. Having realized what I really needed, I began to experiment with the entities known to me at that time in C#.

## Design
The Structural diagram shows the simplest scheme of signal transmission from the sender to the receiver:

![Structural diagram]({{site.url}}/assets/images/posts/{{page.date | date: "%Y-%m-%d"}}-{{page.slug}}/structural-diagram.gif)

The receiver subscribes to a specific type of signal. The sender generates a signal, transmits data to it, and sends it to the "ether". The receiver catches this signal and extracts data from it and processes it. The recipient can break the connection by unsubscribing from the signal. Thus, the sender and the receiver may not know anything about each other, and the signal serves as an interface for transmitting data between them.

## Implementation
I decided to make the base class universal for all signals, while limiting the type parameter to its inheritors:
```csharp
public abstract class Signal<T> where T : Signal<T>
{
    public delegate void Hendler(T signal);
    protected static Hendler hendlers;

    public static void Subscribe(Hendler hendlers)
    {
        Signal<T>.hendlers += hendlers;
    }

    public static void Unsubscribe(Hendler hendlers)
    {
        Signal<T>.hendlers -= hendlers;
    }

    protected static Hendler GetUniqueHendlers(Hendler hendlers)
    {
        HashSet<int> hashs = new HashSet<int>();

        foreach (var hendler in hendlers.GetInvocationList())
        {
            var hash = System.String.GetHashCode(
                hendler.Target?.GetHashCode() + "" +
                hendler.Method.DeclaringType + "" +
                hendler.Method.GetBaseDefinition()
                );

            if (hashs.Contains(hash))
            {
                hendlers -= hendler;
            }
            else { hashs.Add(hash); }
        }
        
        return hendlers;
    }

    public class Exception : System.Exception
    {
        public Exception(string message) : base(message) { }
    }
}
```
Inside it, a static variable is encapsulated in the form of a delegate that stores methods that accept an object derived from the `Signal<T>` type as a prameter. This variable is used to implement the signal subscription mechanism.
> **Attention!** _With great power comes great responsibility:_ since method references are stored in a static variable, make sure that the objects are unsubscribed from the signal in a timely manner, otherwise the garbage collector will not be able to delete them and there will be a memory leak.

Even though the variable is declared in the base class, a delegate is created for each derived class. Thanks to this, the recipient receives only messages of the type to which he subscribed. The `Subscribe(Hendler hendlers)` method is used to add subscriber delegates, and `Unsubscribe(Hendler hendlers)` - to delete them.

### Example of a simple signal
To implement a simple signal, it is enough to inherit from the `Signal<T>` type, passing the type of the derived class to it, and to add a method for sending this signal to the recipients:
```csharp
public class Message : Signal<Message>
{
    private string text;
    public string Text { get => text; }

    private Message(string text)
    {
        this.text = text;
    }

    public static void Send(string text)
    {
        // Calls delegates that accept this type as a parameter
        hendlers?.Invoke(new Message(text));
    }
}
```
In this implementation, the `Message` signal encapsulates a string of text that it will receive when it is created and then send to the recipients.
To send a message, use the static `Send(string text)` method. All it needs to do is create an instance of the `Message` class and call the delegates subscribed to it, passing this object as a parameter.

### Sending and receiving a signal
To send the above signal, a simple line is enough:
```csharp
Message.Send("Hello World!");
```
But before you receive this signal, it is advisable to subscribe to it:
```csharp
Message.Subscribe(Receive);
```
As a parameter, specify the name of the method that handles this signal:
```csharp
private void Receive(Message message)
{
    System.Console.WriteLine(message.Text);
}
```
Do not forget to unsubscribe as soon as the receipt of this signal is no longer relevant for us:
```csharp
Message.Unsubscribe(Receive);
```

### Example of an extended signal
The above signal example has two significant drawbacks:
* The receiver can subscribe to it several times, and when sending a signal, the added method will be called as many times as it was added.
* If no one is subscribed to the signal, the sender will send the signal to the void.

To successfully overcome these problems, you can implement the signal as follows:
```csharp
public class Error : Signal<Error>
{
    private string text;
    private static HashSet<int> hashs = new HashSet<int>();
    
    public string Text { get => text; }

    public Error(string text)
    {
        this.text = text;
    }

    public new static void Subscribe(Hendler hendlers)
    {
        Signal<Error>.hendlers = GetUniqueHendlers(Signal<Error>.hendlers + hendlers);
    }

    public static void Send(string text)
    {
        try
        {
            hendlers.Invoke(new Error(text));
        }
        catch (System.NullReferenceException e)
        {
            throw new Exception(e.Message);
        }
    }
}
```
In this example, when subscribing to a signal, the `GetUniqueHendlers(Hendler hendlers)` method is used to exclude all duplicates from the delegates. As a parameter, we pass the delegates that are already subscribed to the signal along with the new delegates, and at the output we get a set of unique delegates. We store this new delegate in our static variable instead of the old one.

Another innovation is that we do not check the delegate for 'null' before making a call. Instead, we catch the `NullReferenceException` error and throw `Signal<T>.Exception` if it occurs of the class, which we can later process according to the application logic.

### Signal preprocessing
If you inherit from a class derived from `Signal<T>` (for example, from `Error`), then the received signal type can no longer be subscribed, but in this new class you can perform some template pre-processing of the data sent in it, for example, add a prefix:
```csharp
class Warning : Error
{
    private Warning(string text) : base("Warning: " + text) { }
    public new static void Send(string text)
    {
        try
        {
            hendlers.Invoke(new Warning(text));
        }
        catch (System.NullReferenceException e)
        {
            throw new Exception(e.Message);
        }
    }
}
```
When sending this signal, all `Error` subscribers will receive messages already with the _Warning_ prefix.

## Instead of a conclusion
I understand that this method may seem unsafe to many people because of a possible memory leak, but even with a kitchen knife, which most of us use to cut bread for sandwiches, you can cut yourself)

To protect yourself when using signals, I recommend that you subscribe to them only objects that are active before the program exits, or whose lifetime can be accurately determined, and at the end of it unsubscribe from the signal. In the case of objects in **Unity3d**, you can(and should) unsubscribe from the signal in the `onDestroy()` method called when the object is removed from the scene. In the case of standard C#, you should not try to unsubscribe in the destructor or in the `Finalize()` method since it is called by the mustor collector only after removing all references to the object, and as you understand, the reference to the object's method will be stored in a static variable until the program ends.

I have created a separate repository with [examples](https://github.com/{{ site.github.owner_name }}/examples-global-signals) so that you can test them.
You can also use global signals in your project by adding [this repository](https://github.com/{{ site.github.owner_name }}/{{ page.repository }}) as a [submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules):

	git submodule add https://github.com/mofrison/global-signals

In any case, it is up to you to use this method or not. Thanks for your attention :)
