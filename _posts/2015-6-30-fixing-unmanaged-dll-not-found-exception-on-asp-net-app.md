---
layout: post
title: "Fixing Unmanaged DLL Not Found Exception On ASP.Net Application"
category: c#, Azure, Asp.net, Debugging
forreview: false
filename: "2015-6-30-fixing-unmanaged-dll-not-found-exception-on-asp-net-app.md"
---
	
I often deal with `DLL` issues and I think it is good to document how I fix them as there would be many others facing similar issues. I assume that you have a fair knowledge of working of ASP.net application.. :)

I received a StackTrace today which said `System.DllNotFoundException: Unable to load DLL '< dll name >'`.  Here is how I tracked it down and fixed it:

- I looked through the application locally and nothing sounded suspicious, DLL in question was available on the bin directory. 
- This is an [Azure WebRole](https://azure.microsoft.com/en-us/documentation/articles/fundamentals-introduction-to-azure/)  running as a [Azure Cloud Service](http://azure.microsoft.com/en-us/services/cloud-services/). So I looked through the [CSPKG by extracting its contents](http://blogs.msdn.com/b/avkashchauhan/archive/2011/12/11/exploring-windows-azure-package-contents.aspx) and the binary was included.
- Since it is a Cloud Service, I could RDP in to the VM and the binary was available on the SiteRoot\0\bin directory.

So, the binary is available but IIS is unable to locate it. What could be wrong? Let's figure it out.

The tool I use in this kind of situation is [ProcMon from SysInternals](http://live.sysinternals.com/Procmon.exe). If ProcMon is new to you, go through [this quick intro video](https://www.youtube.com/watch?v=pjKNx41Ubxw) by [Scott Hanselman](https://twitter.com/shanselman).

In continuation,

- I copied over ProcMon to the VM on staging environment, which had only one VM running (if there are more VM's you are not sure which VM wold get the reqeust from client...!).
- I set ProcMon to capture file operations of w3wp.exe process and tried to send some requests to the URL which fails. Another level of filter is to make ProcMon show only rows that contain path with the binary in question.
- I could see that IIS is probing for the binary in may directories, which all started with IIS temp directory which looks something like `D:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\root\59a01799\5b6b1a2\assembly\dl3\a78168f6\7cabd0f6_d698d001\< binary name>`. Other directories it probes are system directories and the ones mentioned on the `PATH` environment variable.
- I looked at above the directory and the binary was not there. I copied over the file manually, to the above location and tried to send some requests to IIS.
- I still got same exception in response...! What might still be wrong??? I checked through the ProcMon and now it was looking for some other DLL. Hmm.. this sounds interesting because IIS doesn't say anything about this new binary.
- I guessed that when IIS tries to load the first binary, there would be other dependencies for this binary and IIS is unable invoke certain methods which need such additional binaries and since it is an unmanaged binary, IIS just points to the original binary.
- I used [Dependency Walker](http://www.dependencywalker.com/) to find out dependencies of the binary in question. There were three other binaries, which I copied to IIS temp from Site root. One important thing to remember while using Dependency Walker is, open the binary from the location where the binary exists, it will mark unavailable binaries in yellow color.
- Now, the issue was fixed and the URLs started working fine.

Now, we just know the cause of the issue and know a rough way to fix it. But why would IIS copy all the binaries in to a temp location instead of executing it from site root? Short answer is IIS will copy resources to a directory (called Shadow Copying )which will be used by AppDomain and will have have a lock on the files to save itself from config file modifications and the like. This [blog post](http://www.ipreferjim.com/2012/04/asp-net-appdomains-and-shadow-copying/) has a nice and detailed info on Shadow Copying.

Since we know that the binary is available on site root, we can either add site root path to `PATH` environment variable (binary in question along with all its dependencies are available under SiteRoot directory) or we can just disable Shadow Copying by adding following web.coinfg entry:

{% highlight xml %}

<system.web>
	<hostingEnvironment shadowCopyBinAssemblies="false"/>
</system.web>

{% endhighlight %}

We need to remember that AppDomain will now consider SiteRoot as it's location and modifications to files under SiteRoot would cause application to crash and not a good idea on production environments. But since I always deploy a full CSPKG to production environment and never touch the files on production environment, I am good with this change.

Well, that's it we have fixed it successfully. I have tried to go as detailed as possible, though if you did not understand something, or have confusions, feel free to comment below, I will try to address them.
