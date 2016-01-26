---
layout: post
title: Working with .NET events in C#
comments: true
---

In .NET with C# you can define custom events using the publish/subscriber pattern. Events can be defined and triggered in
a custom fashion, releasing the burden from the developer in the process of monitoring and handling them. In this
example i will demonstrate the use of events for notifying blog followers when a new blog post is made by their
favorite blogger.

The first thing you need to do is create the class that will represent the information passed from the publisher
(in this scenario the blogger) to the subscriber (the blog follower):

```csharp
using System;

namespace Allan.Examples.EventHandlingExemple
{
    /* The event info that will be handed to the subscribber/listener*/
    public class BlogInfoEventArgs: EventArgs
    {
        public BlogInfoEventArgs(string blogPostSumary)
        {
            this.BlogPostSumary = blogPostSumary;
        }

        public string BlogPostSumary { get; private set; }
    }
}
```

Notice here that the event information class should extend System.EventArgs. The info that will be passed 
to the blog follower is a brief summary of the blog post. Next we should define the class that represents
the event publisher:

```csharp
using System;

namespace Allan.Examples.EventHandlingExemple
{
    /* The event info that will be handed to the subscribber/listener*/
    public class BlogInfoEventArgs: EventArgs
    {
        public BlogInfoEventArgs(string blogPostSumary)
        {
            this.BlogPostSumary = blogPostSumary;
        }

        public string BlogPostSumary { get; private set; }
    }
    
    /* The event publisher*/
    public class Blogger 
    {
        public Blogger(string name)
        {
            this.Name = name;
        }

        public string Name { get; private set; }

        public event EventHandler<BlogInfoEventArgs> NewBlogPostInfo;

        public void NewBlogPost(string blogPostSumary)
        {
            Console.WriteLine("Blogger {1}, new blog post: {0}", blogPostSumary, this.Name);
            RaiseNewBlogPostInfo(blogPostSumary);
        }

        protected virtual void RaiseNewBlogPostInfo(string blogPostSumary)
        {

            EventHandler<BlogInfoEventArgs> newBlogPostInfo = NewBlogPostInfo;

            if (newBlogPostInfo != null)
            {
                newBlogPostInfo(this, new BlogInfoEventArgs(this.Name + " blogged: " + blogPostSumary));
            }
        }
    }
}
```

Here we defined a name for our blogger for identification purposes in the case we want to define multiple event sources. The method
NewBlogPost(string blogPostSumary) will be invoked when we choose to raise a new event. NewBlogPostInfo is a delegate
that binds to a method in the subscriber. The line of code invoking newBlogPostInfo is a direct call
to a method that resides in the subscriber, the first parameter is a reference to the publisher and the second parameter
is the object containing information about the event. Next we define the code for the subscriber:

```csharp
using System;

namespace Allan.Examples.EventHandlingExemple
{
    // The event subscriber
    class BlogFollower
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
    }
}
```

Here we define the blog follower with his name and the method that will be called when the 
event is triggered. Now the last but not least, we define our program main entry point
and then make use of the classes just created:

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
            dave.NewBlogPostInfo += allan.ThereIsANewBlogPost;

            // Publish event
            dave.NewBlogPost("Today i found out that roses are red");

            var jonh = new BlogFollower("Jonh");
            dave.NewBlogPostInfo += jonh.ThereIsANewBlogPost;

            // Publish event: now both blog followers will receive the notification
            dave.NewBlogPost("Today i eat pizza, it was very good");

            dave.NewBlogPostInfo -= allan.ThereIsANewBlogPost;

            // Publish event: now allan will not receive the notification
            dave.NewBlogPost("Today i'm going to the beach");
            
            Console.ReadKey();
        }
    }
}
```

Here we first define a blog follower (Allan) and bind him to the event handler in the publisher, the
line dave.NewBlogPostInfo += allan.ThereIsANewBlogPost adds the subscriber method to the event handler delegate
in the publisher, stating that Allan want to receive updates from Dave's blog. Next Dave raises a new event
when he publishes a new blog post. Allan is listening for events from Dave, so his method will be called.
Next a new blog follower is created and binded to Dave, when Dave publishes a new blog post both followers
receive the notification, notice that when newBlogPostInfo
is called at this point it references two methods and both will be called. Next we remove Allan from the event handler delegate, 
when Dave publishes a new blog post only Jonh will be notified.

That is it for the post. In the next post i will talk about the problem with this approach in the case a subscriber isn't in scope
anymore and we want to garbage collect him (even if we remove the subscriber from the event handler delegate, the delegate 
still keeps a strong reference to it). We achieve this by the use of a weak event manager.
