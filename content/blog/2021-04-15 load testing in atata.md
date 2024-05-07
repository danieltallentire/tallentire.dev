+++
title = "Load testing using Atata and Selenoid"
date = 2021-04-15
description = "Solving a UI automation testing problem for user defined forms."
[taxonomies]
tags = ["testing", "selenium", "docker", "automation", "atata"]
+++

I'm Technical Director at [Parker Software](https://www.parkersoftware.com), a company with a long history of making market-leading [live chat](https://www.whoson.com) software and [automation components](https://www.thinkautomation.com).

I'm writing this article as part of CTO West Midlands writing group.

[Atata](https://atata.io) is a great open source project created by @yevgeniyshunevych 

# Contents

[What are we doing](#what-is-load-testing-normally-like-for-whoson)
[Why Atata?](#why-atata-and-not-one-of-the-regular-load-test-systems-like-jmeter)
[The Solution](#the-solution)


# What is load testing normally like for WhosOn?

Normally, we run our load tests in WhosOn using our APIs.  This allows us to simulate the volume of requests coming in to the WhosOn server, but with a cheap and easy setup.
These are normally load tests like "1,000 concurrent chats" or "500 connected operators".

## What is wrong with doing this?

We've recently come across some instances where load has caused a problem, but not to do with the overall capacity. The problem has been caused by behaviours of customers and operators that cause spikes.  These spikes may cause an overload in a single component of the system, or a cascade through the system.
The spike behaviour might not be simple - it may have multiple factors.

If your load test is too simple it doesn't capture the real world behaviour of the operators.

## What behaviour do we want to replicate?

We would like to create a load test that can simulate a shift change using our operator web client.  During shift change, a backlog of queuing chats will build up - this could reach 100 chats in the queue, and as the new shift comes on board, logging in 50-100 users within a minute, those chats are then allocated out to the new operators. During this period, supervisors are also executing a variety of real time reports and pulling metrics against operators who are finishing up their shift, and monitoring the queue lengths to make sure that the active customers get serviced promptly.

# Why Atata and not one of the regular load test systems like JMeter?

- We already use Atata within our testing pipelines. Our devs are familiar with it.
- It allows us to easily vary our load tests, and select tests for different scenarios by altering our coded tests.
- It lets us execute tests in a variety of different browsers.
- We can record screenshots or video the results if we want to see what the experience is like.

# The solution
As part of our Azure DevOps process, we will create a test suite that can be executed for every release.
This will run a two stage process.

1. test the overall chat system with pick up by a bot.
2. test the client login system, with pick up by a UI automated chat agent.

We will use Selenoid to run a cluster of selenium instances in docker containers.  This could be swapped out for any selenium cluster.

An XUnit test will drive the overall process, and a threshold will be set on various metrics to show whether that test was successful or not.

The test will also output a test file reporting on key metrics from the cluster, including how long the chats took to be handled, end-to-end message time and similar.

# The code

I already started an earlier attempt at creating an Atata load test tool a few weeks ago. This was just using a basic set of browser on my box, but I ran into a lot of problems.

Firstly, it killed my computer, trying to pop open new browser windows, and second, it wouldn't be easy to transfer this into the DevOps pipeline.

### Setting up Selenoid
I'm running on windows, so a few hoops to jump through to get Selenoid up and running:

1. update docker
2. change docker to use linux mode 
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4ub4bmzmsa4v67x83cw3.png)
3. force docker to use the correct linux context and permissions
`& 'C:\Program Files\Docker\Docker\DockerCli.exe' -SwitchDaemon`
4. download the correct version of the selenoid configuration manager from https://github.com/aerokube/cm/releases
5. run `./cm.exe selenoid start` to start up selenoid
6. run `./cm.exe selenoid-ui start` to start up the UI so I can see what's happening

Now we have selenoid running, we can open up the selenoid UI on localhost:8080 and see what the connection details are for selenoid:
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mxc6nq64ruqgsbpegfh3.png)

### Building a global context
Based on this, and the Atata documentation, I created a global context driver implementation:

``` csharp
var capabilities = new DesiredCapabilities();
capabilities.SetCapability(CapabilityType.BrowserName, "chrome");
capabilities.SetCapability(CapabilityType.BrowserVersion, "89.0");
var driver = new RemoteWebDriver(new Uri("http://localhost:4444/wd/hub"), capabilities);

AtataContext.GlobalConfiguration
    .UseDriver(driver);
```

### Creating a runner class
Since I need to run multiple Atata instances, I don't want to launch Atata as normal from a test - I need to create some tasks that can run Atata within them and maintain a separate context.

``` csharp
AtataContext.ModeOfCurrent = AtataContextModeOfCurrent.AsyncLocal;
```
This line of code instructs Atata to ensure that each asynchronous code block has its own instance of the Atata context.

#### Building the context
``` csharp
private void CreateContext()
{
    _context = null;
    var contextBuilder = AtataContext.Configure();
    _context = contextBuilder.Build();
}

public async Task SetupContext()
{
    await Task.Run(() =>
    {
        CreateContext();
        finishTime = DateTime.Now.AddMinutes(2);
    });
}
```
This creates the individual context for each LoadTestRunner

#### Running the test
``` csharp
public async Task Run(CancellationToken cancellationToken)
{
    AtataContext.Current = _context;
    try
    {
        var chatPage = Go.To<Survey>(url: "http://192.168.1.70/newchat/chat.aspx?domain=www.test.com")
            .VisitorName.SetRandom()
            .StartChat.ClickAndGo<Chatting>();

        while (!cancellationToken.IsCancellationRequested)
        {
            if (finishTime <= DateTime.Now)
            {
                chatPage.CloseWindowButton.Click();

                return;
            }

            await Task.Delay(10000);

            chatPage.ChatReply.SetRandom()
                .SendButton.Click();
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.ToString());
    }
}
```
This does a really simple start chat function, with only filling in the name field, sending a message every ten seconds, and closing after two minutes.

## Time to try it out
I gave it a run, with a single session executing.
Success! It created a session in selenoid, and completed the test successfully - I got a chat popping up.

OK - time to up it to five sessions.

_FAIL_ it only launched one session, and I got a ton of errors.

### What went wrong?
Digging in to the debugging, I found that it is only launching one instance of the driver, not multiple as expected.

I changed the global code to:
``` csharp
var capabilities = new DesiredCapabilities();
capabilities.SetCapability(CapabilityType.BrowserName, "chrome");
capabilities.SetCapability(CapabilityType.BrowserVersion, "89.0");

AtataContext.GlobalConfiguration
    .UseDriver(() =>
    {
        return new RemoteWebDriver(new Uri("http://localhost:4444/wd/hub"), capabilities);
    });
```
This now lets me launch 5 simultaneous sessions.

## Can it go further?
I played around with parameters to reduce the memory and CPU usage, but the most I got working was 10 sessions - this partly seems to be to do with my IIS as well - something internally is preventing the sessions from connecting.  Indeed, during the test my box prevented a manual chat session request from loading. My cpu was maxed out, and the memory usage was very high.  This should work better in the cloud.

This is the selenoid setting I used to create a new instance with limited CPU and memory usage:
`./cm.exe selenoid start --vnc --tmpfs 248m -g "-limit 20 -mem 128 -cpu 1.0"`

# What is next?

For Part 2, I'm going to explore moving this into our Azure DevOps Pipeline.

I need to optimise the memory usage further as well to ensure we can use more of these instances for a quick burst of testing.
