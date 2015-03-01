---
layout: post
title: "C# component for prism code highlighting"
description: "C# component added for prism code highlighting"
category: "front-end"
tags: ["c#", "prism", "code-highlighting", "syntax-highlighting"]
---
{% include JB/setup %}

Looking for a good code syntax highlighting tool for web, I stumbled upon [prism](http://prismjs.com/) and I found it simple, fast and flawless (as far as my needs go).

What makes it particularly cool is that [it is open source](https://github.com/LeaVerou/prism) and superbly extensible.

So I added a component (you can read 'plugin') for C# to it's list of components. You can add any missing language by simply contributing to the [github repository](https://github.com/LeaVerou/prism).

You can download the prism from [here](http://prismjs.com/download.html) and choose C# (alongside other languages).

There are various themes to choose from. 
here is an example of how your code will look like in 'okaidia' theme:

<!--more-->

### Standard code

<pre><code class="language-csharp">
using System;
class Person
{
    private string myName ="N/A";
    private int myAge = 0;

    // Declare a Name property of type string:
    public string Name
    {
        get 
        {
           return myName; 
        }
        set 
        {
           myName = value; 
        }
    }

    // Declare an Age property of type int:
    public int Age
    {
        get 
        { 
           return myAge; 
        }
        set 
        { 
           myAge = value; 
        }
    }

    public override string ToString()
    {
        return "Name = " + Name + ", Age = " + Age;
    }

    public static void Main()
    {
        Console.WriteLine("Simple Properties");

        // Create a new Person object:
        Person person = new Person();

        // Print out the name and the age associated with the person:
        Console.WriteLine("Person details - {0}", person);

        // Set some values on the person object:
        person.Name = "Joe";
        person.Age = 99;
        Console.WriteLine("Person details - {0}", person);

        // Increment the Age property:
        person.Age += 1;
        Console.WriteLine("Person details - {0}", person);
    }
}
</code></pre>

### Some code with contextual keywords

<pre><code class="language-csharp">
    /// &lt;summary>
    /// Some documentation
    /// &lt;/summary>
    partial class MyClass
    {
        public static SomeMethod(out int something){
            var query = from item in new[]{ "a", "b", "c" }
                        where item.Length > 0
                        orderby item
                        select item;
        }

        public IEnumerable<string> Iterator()
        {
            for (int i = 0; i < length; i++)
			{
                yield return i.ToString();
			}
        }
    }
</code></pre>
