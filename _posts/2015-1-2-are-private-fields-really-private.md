---
layout: post
title: "Are private fields really private?"
category: c#, Reflection
forreview: false
filename: "2015-1-2-are-private-fields-really-private.md"
---

As the saying goes, private fields and properties of a class are not available outside the class. But are they really private? Well, they are not; once they are in memory, anyone can tap in to them.

Let's take a look at the code:

{% highlight csharp %}
void Main()
{
	//create an instance and dump
	var demo = new PrivateDemo().Dump();
	
	FieldInfo[] privateFieldInfos = typeof(PrivateDemo).GetFields(BindingFlags.NonPublic | BindingFlags.Instance);
	
	//set new value and dump
	foreach (var fieldInfo in privateFieldInfos)
	{
		fieldInfo.SetValue(demo, "I am no longer private...");
		fieldInfo.GetValue(demo).Dump(fieldInfo.Name);
	}
}

public class PrivateDemo
{
	public string PublicProperty { get; set; }
	private string PrivateProperty { get; set; }

	private string privateField;
	
	public PrivateDemo()
	{
		PublicProperty = "I am public...";
		PrivateProperty = "I am private...";
		privateField = "I am also Private...";
	}
}
{% endhighlight %}

And output from [LinQPad][1]:

![](https://raw.githubusercontent.com/amithegde/amithegde.github.io/master/contents/img/2015-1-2-accessing-private-fields-output.jpg)

[1]:http://www.linqpad.net/