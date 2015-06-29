---
layout: post
title: "Finding conflicting binary versions"
category: c#, Reflection
forreview: false
filename: "2015-2-25-finding-conflicting-binary-versions.md"
---

I deal with lot of .Net assemblies on a project I work with. Assemblies change their version and their dependencies frequently and one very common exception I see is:

`Could not load file or assembly '...., Version=...., Culture=neutral, PublicKeyToken=....' or one of its dependencies. The located assembly's manifest definition does not match the assembly reference.`

We can handle this by using [Assembly Redirection][1] as long as there are no method signature changes. Adding assembly redirection can cause other side effects in some cases. If we want to change assemblies to reference same version of dependency,  finding which assemblies refer this particular binary and which versions they refer becomes tricky.

Let's write some code in C# to solve this:

{% highlight csharp %}

void Main()
{
	var folder = @"";
	var files = Directory.GetFiles(folder,"*.dll",SearchOption.AllDirectories);
	
	var searchForNamespace = "";//Eg: Newtonsoft.Json
	
	if(files.Any())
	{
		foreach (var element in files)
		{
			try
			{	        
				var assembly = Assembly.LoadFile(element);
				if(assembly != null)
				{
					var references = assembly.GetReferencedAssemblies();
					if(references.Any(x => x.FullName.StartsWith(searchForNamespace)))
					{
						var referencedAssembly = references.Where(x => x.FullName.StartsWith(searchForNamespace));
						Console.WriteLine ("FileName: "+ element);
						Console.WriteLine ("Assembly: " + assembly.FullName + "\nReferenced Assembly: ");
						referencedAssembly.Dump();
					}
				}
			}
			catch (Exception ex)
			{
				//element.Dump("exception loading");
			}
		}
	}
}

{% endhighlight %}

This code is written for [LinQPad][2]. If you want to execute it on VS, LinqPad extension method `.Dump()` needs to be replaced.

[1]:https://msdn.microsoft.com/en-us/library/7wd6ex19%28v=vs.110%29.aspx
[2]:http://www.linqpad.net/