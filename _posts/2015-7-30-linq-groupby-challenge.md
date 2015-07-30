---
layout: post
title: "LINQ Challenge #1"
category: c#, LINQ
forreview: false
filename: "2015-7-30-linq-groupby-challenge.md"
---

In this challenge let us explore [LINQ GroupBy](https://msdn.microsoft.com/en-us/library/vstudio/bb534304%28v=vs.100%29.aspx) based on multiple properties.

The problem is to Group By Property1 if it matches certain condition or Property2 and collect a meaningful key for each key. As earlier, I use [LinqPad](http://www.linqpad.net/) to get the code working quickly.

{% highlight csharp %}
void Main()
{
	var list = new List<Person> {
		new Person { Parent1 = "SomeName1", Parent2 = "asdconst1fgh", Age = 10 },
		new Person { Parent1 = "SomeName2", Parent2 = "asdconst2fgh", Age = 10 },
		new Person { Parent1 = "SomeName1", Parent2 = "asdconst1fgh", Age = 10 } };

	var groupTypes = new List<string> { "Const1", "Const2" };

	//query syntax
	var gQuery1 = from data in list group data by GetKey(data, groupTypes);
	//method syntax
	var gQuery2 = list.GroupBy(data => GetKey(data, groupTypes));

	gQuery1.Dump();
	gQuery2.Dump();
}

//returns a meaningful key for each group
private string GetKey(Person data, List<string> groupTypes)
{
	//condition 1
	if (data.Parent1 == "SomeName1")
	{
		return data.Parent1;
	}
	
	//or condition 2
	return groupTypes.First(x => data.Parent2.ContainsOrdinalIgnoreCase(x));
}

//sample model
public class Person
{
	public string Parent1 { get; set; }
	public string Parent2 { get; set; }
	public int Age { get; set; }
}

public static class Extensions
{
	public static bool ContainsOrdinalIgnoreCase(this string str, string value)
	{
		return str.Contains(value, StringComparison.OrdinalIgnoreCase);
	}

	/// <summary>
	/// Performs a contains and takes a comparisonType (allows case insensitive comparison)
	/// </summary>
	public static bool Contains(this string str, string value, StringComparison comparisionType)
	{
		return !string.IsNullOrWhiteSpace(value) && str.IndexOf(value, comparisionType) >= 0;
	}
}
{% endhighlight %}

The crux of the problem lies in the method `GetKey` which returns a meaningful key for each row based on either `Parent1` or `Parent2`.