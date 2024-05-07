+++
title = "Using Atata to create UI tests for user defined web forms"
date = 2021-03-26
description = "Solving a UI automation testing problem for user defined forms."
[taxonomies]
tags = ["testing", "selenium", "csharp", "atata"]
+++


# The Problem
At Parker Software, our chat software allows for customers to create whatever inputs they want to show on the pre-chat survey, which is asked before a user enters the chat:
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pkavq1i5zsifu2gv8ib1.png)

These fields can be any html field type, including simple text inputs, password fields, radio buttons and drop down lists.

When we setup a new theme for a customer, we want to ensure that the layout and behaviour remains the same when any updates and modifications to the core chat window take place. At the moment this is a manual process whereby the upgrade is performed, and a QA team member will run through the chat window, inspecting it from errors.  This manual process is somewhat error prone.

**I want to create an automated solution that can work regardless of the fields that have been set in the drop down**

We use Atata to do some of our other automated UI testing, and I wanted to have a go at using it to solve this problem. 

# What is Atata?
[Atata](https://atata.io) is an automated testing framework based on Selenium.  You can use C# code to define pages as class files, then really simply set up a wide variety of tests without needing to worry about what the client side looks like.

Thanks to Yevgeniy Shunevych for making Atata so awesome and putting out continuous updates and improvements.

## What does it look like?

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8ed54bv4ak803i388dw0.png)
 
## What are the challenges?

Atata doesn't have a "dynamic" control built in - it usually requires all elements that you want to interact with to be predefined, so I will need to spend some time working on resolving that.

## The setup

I'm using Atata with XUnit - we use XUnit for all our unit testing, so Atata has to run through that - it is a little more difficult as Atata doesn't have a native XUnit adapter, but only a few extra changes are needed to integrate with XUnit's logs.

# The code
You can get the code at https://github.com/danieltallentire/AtataDynamicForms

## Attempt 1
First, I attempted to use the ControlList class with the built in Input control.

``` c#
using Atata; 
 
namespace AtataDynamicFormTester 
{ 
    using _ = ChatStartPage; 
 
    [WaitForLoadingIndicator] 
    class ChatStartPage : Page<_> 
    { 
        [ControlDefinition(ContainingClass = "input-group")] 
        public ControlList<Input<string, _>, _> SurveyFields { get; private set; } 
 
        public Button<Chatting, _> StartChat { get; private set; } 
 
        public _ SetRandom() 
        { 
            foreach (var field in SurveyFields) 
            { 
                field.SetRandom(); 
            } 
 
            return this; 
        } 
    } 
} 
```

Each control on the form is wrapped in a div with the input-group class. This means we can use it with the list locator to find each of the inputs inside.

This came unstuck pretty quickly. It was able to detect and find the elements in the control list, but unable to set a value on them, even with a simple text field.

## Attempt 2
My next attempt was to use a simple custom control to wrap the input control. In Atata a custom control is a class derived from the parent Control.

The containing class now moves to part of the control definition. 
We can rely on this element existing, so we can iterate the fields.

``` c#
    [ControlDefinition(ContainingClass = "input-group")] 
    class DynamicControl<TOwner> : Control<TOwner> where TOwner : PageObject<TOwner> 
    { 
        [FindByClass("form-control")] 
        public Input<string, TOwner> TextInput { get; set; } 
 
        public void SetRandom() 
        { 
            TextInput.SetRandom(); 
        } 
    } 
```
The list is then changed to reference it: 
``` c#
  [ControlDefinition(ContainingClass = "input-group")] 
        public ControlList<DynamicControl<_>, _> SurveyFields { get; private set; } 
```
_Success!_

Well... some success. This worked fine for simple fields like Name and Company.  It got unstuck when it reached an email address, trying to fill it as a random field, and completely ignored the other controls.

My next thing to change was to try to switch based on the input value:

``` c#
public void SetRandom() 
{ 
	switch (Input.Attributes.Type) 
	{ 
		case "text": 
			Input.SetRandom(); 
			break; 
		case "email": 
			Input.Set(Atata.Randomizer.GetString("{0}@{0}.com")); 
			break; 
		case "date": 
			Input.Set("03/03/2021"); 
			break; 
		case "checkbox": 
			Input.Set("true"); 
			break; 
		default: 
			// ignore 
			break; 
	} 
} 
```
OK. Getting closer - this worked nicely for the email address and the date - although its a bit clunky setting the date as a string.
Checkbox didn't work at all, and the select wasn't even detecting.

## Attempt 3
Looking again at the classes, I'd forgotten that *select* elements aren't *input* elements... d'oh!

I had a play around, and found that I could add multiple elements in to the class:

``` c#
        [FindByClass("form-control")] 
        public Input<string, TOwner> Input { get; set; } 
 
        [FindById("fld")] 
        [TermFindSettings(TargetAttributeType = typeof(FindByIdAttribute), Match = TermMatch.StartsWith)] 
        public CheckBox<TOwner> Checkbox { get; set; } 
 
        [FindByClass("form-control")] 
        public Select<TOwner> Select { get; set; } 
```

And it wouldn't error out for the missing ones in the cases where the control didn't contain those elements, however if I tried to access the control _Input_ on a select element it would throw an exception (rightly, because the Input doesn't exist).

I found that I could create a check to see if the element was found by checking for the Exists property with the IsSafely flag set:
``` c#
 if (Checkbox.Exists(new SearchOptions() { IsSafely = true })) 
```

This caused a slowdown, so this lead me to the final code with a predefined timeout:

``` c#
 [ControlDefinition(ContainingClass = "form-group")]
    class DynamicControl<TOwner> : Control<TOwner> where TOwner : PageObject<TOwner>
    {
        [FindByClass("form-control", Timeout = 0.01)]
        public Input<string, TOwner> Input { get; set; }

        // checkboxes aren't wrapped by a nice form-control wrapper, so have to use a different search method
        [FindById("fld", Timeout = 0.01)]
        [TermFindSettings(TargetAttributeType = typeof(FindByIdAttribute), Match = TermMatch.StartsWith)]
        public CheckBox<TOwner> Checkbox { get; set; }

        [FindByXPath(".//*[contains(concat(' ', normalize-space(@class), ' '), ' form-control ')]/descendant-or-self::input[@type='date']", Timeout = 0.01)]
        public DateInput<TOwner> DateInput { get; set; }

        [FindByClass("form-control", Timeout = 0.01)]
        public Select<TOwner> Select { get; set; }

        public void SetRandom()
        {
            if (Checkbox.Exists(new SearchOptions() { IsSafely = true }))
            {
                Checkbox.Set(true);
            }
            else if(Select.Exists(new SearchOptions() { IsSafely = true }))
            {
                Select.Set(Select.Options[Atata.Randomizer.GetInt(0, Select.Options.Count - 1)].Value);
            }
            else if (DateInput.Exists(new SearchOptions() { IsSafely = true}))
            {
                DateInput.SetRandom();
            }
            else if (Input.Exists(new SearchOptions() { IsSafely = true }))
            {
                switch (Input.Attributes.Type)
                {
                    case "text":
                        Input.SetRandom();
                        break;
                    case "email":
                        Input.Set(Atata.Randomizer.GetString("{0}@{0}.com"));
                        break;
                    default:
                        // ignore
                        break;

                }
            }           
        }
    }
```

I needed to use a manual XPath search for the date input - this was because Atata by default allows for date inputs to be a normal text input with a mask, so was also picking up the default text boxes as date fields too.

I had to add a new randomizer to make the DateInput SetRandom work.
Now I've got the chat window starting nicely.

The containing class was actually incorrect in my earlier examples, and I changed it to form-group - this was caused by the checkboxes not being inside an input-group div.

Just enabling screenshotting in the places I want them:

``` c#
        public void FillPrechatSurvey()
        {
            Go.To<ChatStartPage>(url: "http://localhost/newchat/chat.aspx?domain=www.parkersoftware.com")
                .Report.Screenshot("Start Chat")
                .SetRandom()
                .Report.Screenshot("Filled Out")
                .StartChat.ClickAndGo()
                .Report.Screenshot("Chatting")
                .Wait(5)
                .CloseWindowButton.Click()
                .Report.Screenshot("Finished");
        }
```


And the results:

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q9o06cftjnpkyqmvmv90.png)
 

# What next?

In order to make this good for production I've got to do some little tweaks - making the resolution and the URL something that can be passed in as an environment variable so it can be run inside DevOps.  This is reasonably straight forward.

In the longer term, I want to be able to configure some built in routes and selections to validate that the data sent in the set random is what is received by the server.  The WhosOn Echo Bot can send the data back, but I'll have to save the output somehow with SetRandom.

## Update
Yevgeniy suggested some refactorings to make my rough code a little neater:

``` c#
 [ControlDefinition(ContainingClass = "form-group")]
    [FindFirst(TargetAllChildren = true)]
    class DynamicControl<TOwner> : Control<TOwner> where TOwner : PageObject<TOwner>
    {
        // checkboxes aren't wrapped by a nice form-control wrapper, so have to use a different search method
        [FindById(TermMatch.StartsWith, "fld")]
        public CheckBox<TOwner> Checkbox { get; set; }

        public DateInput<TOwner> DateInput { get; set; }
        
        public EmailInput<TOwner> EmailInput { get; set; }
        
        public Select<TOwner> Select { get; set; }
        
        public TextInput<TOwner> TextInput { get; set; }
        
        public void SetRandom()
        {
            if (Checkbox.IsPresent)
            {
                Checkbox.Set(true);
            }
            else if(Select.IsPresent)
            {
                Select.Set(Select.Options[Atata.Randomizer.GetInt(0, Select.Options.Count - 1)].Value);
            }
            else if (EmailInput.IsPresent)
            {
                EmailInput.Set(Randomizer.GetString("{0}@{0}.com"));
            }
            else if (TextInput.IsPresent)
            {
                TextInput.SetRandom();
            }
            else if (DateInput.IsPresent)
            {
                DateInput.SetRandom();
            }
                    
        }
    }
```

Changing the inputs to use IsPresent is much more succinct.
Adding the top level attribute for TargetAllChildren means that the class name isn't required.

Using TextInput instead of date input means that the order can be changed to allow a better ordering, and get the correct text or date input.

For this to work I had to add an EmailInput as well to catch type='email'



