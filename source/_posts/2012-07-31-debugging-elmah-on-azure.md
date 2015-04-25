---
title: Debugging Elmah on Azure
subtitle: "What do you do when the error handler is erroring..."
date: "2012-07-31T12:00:00"
categories: 
    - Development
tags:
    - SQL
    - Azure
---

Recently I was deploying an application to the new Azure Websites service, and I noticed a slight problem with my app.

ELMAH worked on my machine, but not in the cloud.

Needless to say, if your exception logging isn't working, then you're probably not going to sleep well at night. So my first question, naturally, was "How in the world do I see what is going on inside ELMAH?".

One thing that many folks don't realize until they need to do something like this is that logging libraries tend to "eat" many of their internal exceptions since they are usually the layer that is added to take care of exactly that. I say they "eat" their exceptions not in a strict sense, but more so because there really is no where you can do a "TRY CATCH" to see what is happening.

##Tracing to the Rescue

Instead of throwing exceptions both ELMAH and Log4net output their exceptions to the System.Diagnostics.Trace object. This is sweet when debugging locally, as any application traces will automatically write to the Output window in Visual Studio. But as the title of this post reads, that doesn't help me at all.

I needed to figure out how to output the traces of my app running in Azure to some storage that I could then view. I did have Log4net working, and it was writing to an SQL Azure database, so I decided to go with that option.

First, I had to learn a few things about ASP.NET and Tracing. ASP.NET has its own tracing mechanism that is somewhat separate from the Diagnostics Tracing (which you can read about here). I could try to write the Diagnostic Trace messages out to the ASP.NET Page Trace mechanism, but in MVC (which I'm using), it doesn't really work. So, honestly my best bet is to write to a database (as writing to a file in the cloud is not very reliable) using a TraceListener class setup to redirect Trace messages to Log4net's log.

My TraceListener looked like this:

```csharp
using System.Diagnostics

public class Log4netTraceListener : TraceListener
{
    private static readonly log4net.ILog _log;

    public Log4netTraceListener()
    {
        _log = log4net.LogManager.GetLogger(typeof(Log4netTraceListener));
    }

    public override void Write(string message)
    {
        if (_log != null)
        {
            _log.Debug(message);
        }
    }

    public override void WriteLine(string message)
    {
        if (_log != null)
        {
            _log.Debug(message);
        }
    }
}
```

Now, I first tried adding my Log4netTraceListener in the web.config. However, this Listener is initialized and added to the "Trace.Listeners" collection before Log4net has initialized in my web application. So instead, I have to add it programmatically during the application startup. I did the following in my Global.asax.cs:

```csharp
System.Diagnostics.Trace.Listeners.Add(new Log4netTraceListener());
```

After all of that, I finally started seeing the following in my Log4net log:
> System.Data.SqlClient.SqlException (0x80131904): Tables without a clustered index are not supported in this version of SQL Server. Please create a clustered index and try again.

Whoa, that is not something I expected to see. Apparently the "ELMAH_Error" table has a non-clustered primary key and no clustered index on the table (not really sure why). So we have two options:
1. Add another column to the table and setup a clustered index on that column.
2. The "Sequence" column is already an "int IDENTITY(1,1)" column - so just setup a clustered index on that column.

I chose option 2. And here's the change to apply after executing the default [ELMAH SQL Script](https://code.google.com/p/elmah/source/browse/src/Elmah/SQLServer.sql):

```sql
CREATE CLUSTERED INDEX [IC_Elmah_Sequence] on dbo.[ELMAH_Error] (Sequence)
```