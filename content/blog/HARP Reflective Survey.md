+++
title = "HARP Reflective Survey"
date = 2023-02-01
description = "Why I built a quick app for a survey."
[taxonomies]
tags = ["MBA"]
+++

I started out with a piece of paper given to me on the MBA.
A questionnaire about my research philosophy. How do my own views on the world influence how I perform research?

The questionnaire was interesting, but being an engineer, and liking a good diagram, I expected to find a website that allowed me to do this survey online, and get a pretty picture out of the other side.

No luck. I'll have to make something myself.
 
I found a [good article by Saunders & Bristow](https://www.researchgate.net/publication/370765449_Heightening_awareness_of_research_philosophy_the_development_of_a_reflexive_tool_for_use_with_students_Heightening_awareness_of_research_philosophy_the_development_of_a_reflexive_tool_for_use_with_stu) on their development of the paper tool for use with students. I used this as the basis for what the questionnaire should look like.

## What does it need to do?

- Show a series of 30 statements
- Allow the user to pick on a scale between "Strongly agree" and "Strongly disagree" for each statement
- Allow to move back and correct an answer
- Categorise the questions by two dimensions - the type of statement, and the research philosophy it supports.
- Show an image of a radar chart that can be copied in to a report

I also needed to be able to build it quickly using tools that I knew how to use, with minimal effort.

## What did I build?

I made a Vite app using Vuejs. I used graph.js for creating the images.
I ended up making something that could generically handle any similar questionairre

## How is it deployed?

I deployed it to [Vercel](https://vercel.com). In order to make it appear as a subfolder, I setup a [rewrite rule on my tallentire.dev deployment](https://github.com/danieltallentire/tallentire.dev/blob/d4d72ddead6f9b859a5695ad536fc01838354649/vercel.json).

This JSON file rewrites requests to /harp/ to the index page of the vercel app, and all assets are rewritten below that folder. 

```json
{
    "rewrites": 
    [
        { "source": "/harp/", "destination": "https://vuejs-harp-reflexive-survey.vercel.app/index.html" },
        { "source": "/harp/:asset*", "destination": "https://vuejs-harp-reflexive-survey.vercel.app/:asset*" },
        { "source": "/(.*)", "destination": "/" }
    ]
}
```


# What about my results

![My research philosophy](../research_philosophy.png)

I lean most heavily towards pragmatism. This means I focus heavily on identifying the problem, and how to practically resolve the problem.
This can make certain types of evaluative study harder for me - for instance I find it difficult to write literature reviews, as it is important to synthesise and understand the information without trying to fix or augment the information from the literature.