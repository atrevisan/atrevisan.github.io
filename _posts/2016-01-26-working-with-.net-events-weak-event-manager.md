---
layout: post
title: Working with .NET events in C# (Weak event manager)
comments: true
---

As i stated in my [last post] (http://atrevisan.tk/2016/01/22/working-with-.net-events/). When working
with events we need to address the situation when our event subscriber is out of scope but is
still referenced by the event handler, in that case it will sit forever in the memory and never
will be garbage collected. To solve this issue we can use a weak event manager so that we don't have
to add/remove subscribers directly to the event handler delegate. The code looks like this:

```csharp
using System.Windows;

namespace Allan.Examples.EventHandlingExemple
{
    public class WeakBlogPostInfoEventManager: WeakEventManager
    {
        public static WeakBlogPostInfoEventManager CurrentManager
        {
            get
            {
                var manager = GetCurrentManager(typeof(WeakBlogPostInfoEventManager))
                    as WeakBlogPostInfoEventManager;

                if (manager == null)
                {
                    manager = new WeakBlogPostInfoEventManager();
                    SetCurrentManager(typeof(WeakBlogPostInfoEventManager), manager);
                }
                return manager;
            }
        }

        public static void AddListener(object source, IWeakEventListener subscriber)
        {
            CurrentManager.ProtectedAddListener(source, subscriber);
        }

        public static void RemoveListener(object source, IWeakEventListener subscriber)
        {
            CurrentManager.ProtectedRemoveListener(source, subscriber);
        }

        void Blogger_NewBlogPostInfo(object sender, BlogInfoEventArgs e)
        {
            DeliverEvent(sender, e);
        }

        protected override void StartListening(object source)
        {
            (source as Blogger).NewBlogPostInfo += Blogger_NewBlogPostInfo;
        }

        protected override void StopListening(object source)
        {
            (source as Blogger).NewBlogPostInfo -= Blogger_NewBlogPostInfo;
        }
    }
}
```

Don't forget to include a reference to the WindowsBase assembly to your project because the
WeakEventManager class that we inherited resides there. Here we defined AddListener and
RemoveListener which parameters are the event source and IWeakEventListener (which we should
implement in our subscribber). StartListening and StopListening need to be overrided, they are
invoked by the framework when the first subscriber is added and the last subscriber is removed
respectively, you can add multiple event sources by testing and casting the source object. Next we
implement the IWeakEventListener interface to our subscriber:

```csharp
using System;
using System.Windows;

namespace Allan.Examples.EventHandlingExemple
{
    // The event subscriber
    class BlogFollower : IWeakEventListener
    {
        private string name;

        public BlogFollower(string name)
        {
            this.name = name;
        }

        public void ThereIsANewBlogPost(object sender, BlogInfoEventArgs e)
        {
            Console.WriteLine("Hey {0}, there is a new blog post from {1}. {2}", 
                              this.name, (sender as Blogger).Name, e.BlogPostSumary);
        }

        public bool ReceiveWeakEvent(Type managerType, object sender, EventArgs e)
        {
            ThereIsANewBlogPost(sender, e as BlogInfoEventArgs);
            return true;
        }
    }
}

```

Here we implemented the ReceiveWeakEvent method that invokes the desired method in the subscriber.
Finally we define out program main entry point (we don't need to modify to publisher code):

```csharp
using System;

namespace Allan.Examples.EventHandlingExemple
{
    class Program
    {
        static void Main(string[] args)
        {
            var dave = new Blogger("Dave");

            var allan = new BlogFollower("Allan");
            WeakBlogPostInfoEventManager.AddListener(dave, allan);
     
            // Publish event
            dave.NewBlogPost("Today i found out that roses are red");

            var jonh = new BlogFollower("Jonh");
            WeakBlogPostInfoEventManager.AddListener(dave, jonh);

            // Publish event: now both blog followers will receive the notification
            dave.NewBlogPost("Today i eat pizza, it was very good");

            WeakBlogPostInfoEventManager.RemoveListener(dave, allan);

            // Publish event: now allan will not receive the notification
            dave.NewBlogPost("Today i'm going to the beach");
            
            Console.ReadKey();
        }
    }
}

```
Notice that the code works as before but now is the weak event manager job to add
and remove event subscribers, enabling garbage collection for unreferenced objects. 
In the next post i will show a easier way to deal with the weak event manager so
you don't need to implement anything. It is possible through the use of the
generic version of the weak event manager.
