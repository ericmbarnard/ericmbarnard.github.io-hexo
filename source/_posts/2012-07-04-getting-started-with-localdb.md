title: "Getting Started with LocalDB"
date: 2012-07-04 17:12:06
categories: 
    - Development
tags:
    - SQL
---
With the launch of SQL Server 2012, a new product called "LocalDB" was launched. This new toy is a great fit for needs that fall somewhere in the space between SQL Express and SQL Compact.

While you can read the full list of features for LocalDB, I'll highlight a few below:

* Easy, and simple MSI to install. I've had very little problems with this, and it is light-years better than trying to setup SQL Express
* Ability to control user access to the database
* Supports virtually all features of SQL Express (Stored Procedures, Triggers, ... even some basic Replication)
* Not an extra service on your machine to manage

Initially, there was some confusion as to whether LocalDB was high-powered file-based database or just a dumbed-down version of SQL Express. I'm not going to debate the semantics of what it is exactly, but I will tell you that you get your standard ".mdf" and ".ldf" files with LocalDB. However, the MSI installs some central libraries and an engine that power the LocalDB instances. The engine does have to have some knowledge as to the existence of any instances of LocalDB in order for those to work. So if you were looking for something where you can just drop a file into a directory and it will just work... not quite, but close.

##Instances
LocalDB has the notion of "Automatic" and "Named" instances. Per version of LocalDB installed on your machine, there is only one "Automatic" instance per user. You can think of this as the default "SQLEXPRESS" instance that is installed when you first download and install SQL Express on a machine, only it is unique per user on the machine. So if you create an application that creates and uses the "Automatic" instance of LocalDB and this application is installed on a machine with 3 different user accounts, you will have 3 different ".mdf" and ".ldf" files after each user logs in and uses the application the first time.  This "Automatic" instance is:
* "Public" – meaning any application process under the user's account can access the instance when logged into a machine
* Accessed via the connection string "(localdb)\{major version}" – which for right now means you'll use "(localdb)\v11.0"

{% img /images/getting-started-with-local-db-01.png %}

* Great for development purposes when you need to just throw together a DB and ship something

##Named Instances
The "Named" instance of LocalDB refers to private instances of a database that you can setup. These are useful if you want an application to have a separate, private database per user on a machine (a "Named" instance will run in a separate process under the current user on a machine).

##Sharing Instances
So what if you want to allow multiple users on a machine to have access to a single LocalDB database instance? Well you can also "share" a "Named" instance of LocalDB. You can control who the instance is shared with, and you can enable/disable sharing at any time.

When connecting with a shared instance, your connection string will need to include an extra ".\" in the server portion. Going with the example above, one would use: "(localdb)\.\MySharedInstance". This tells the engine that this is a shared instance.

##SqlLocalDb Utility
When you install LocalDB on a machine, it also installs the SqlLocalDb command line utility. It's pretty simple to use, and allows you to setup batch files to control the creation of your LocalDB's. One of the first things I do is create a "SetupDb.cmd" file in my source code folder of an app I'm building. This allows other devs to easily replicate setting up a development database on their machine in the same way.

{% img /images/getting-started-with-local-db-02.png %}

##Where Does Everything Go?
One of my first questions was, where are the actual database files?

> The default is: "C:\Users\{USER}\AppData\Local\Microsoft\Microsoft SQL Server Local DB\Instances"

If I've created the "Northwind" Named instance as above, and I go to the default folder, I will see:

{% img /images/getting-started-with-local-db-03.png %}

Inside each of these folders are your typical "master.mdf" , "temp.mdf", log files, etc... If you want to create your database's ".mdf" and ".ldf" files in a separate directory, you can easily do that by specifying the location when running a T-SQL statement to "CREATE DATABASE ...". Specifying where your data lives is not really any different than what you've done with SQL Server in the past.

If you want to point your application to specific '.mdf" or ".ldf" files at runtime, you can specify that in your connection string using the "AttachDbFileName" attribute, you can also read more [here](https://msdn.microsoft.com/en-us/library/hh309441.aspx).