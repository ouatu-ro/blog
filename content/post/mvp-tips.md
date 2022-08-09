---
title: "MVP Tips"
date: 2022-03-28T00:29:54+03:00
draft: false
---

# Some context
I had a visualisation app I built for exploring if our aspect based sentimet analysis (ABSA) pipeline. The image below explains both what ABSA is and what the app did.
![ABSA image](/blog/absa-example.png)

Some people (backend people) in my company needed to explore text messages to see what problems users complain about and how many messages they write about each problem. So I took the visualisation app and the ABSA pipeline and just ran it on the new dataset. People were very hapy with this solution.

Another set of people (UX people) wanted the exact same thing, so I followed the exact same steps.
This time I said that I'll first sit with them to see how they use the app. I sit down with them one by one and let them navigate the app. One after another tried to click on visualisations to no prevail, struggled with what a named entity or a random sample is, ignored full pages because there were no visualisations by default.

# What I\'ve learned

Don't give users the responsibility of finding the optimal way to solve their problem under the pretext of flexibility (they literally told me this). Put it another way, be opinionated.

Don’t try to solve more than one problem. Put it another way, be opinionated.

Get fast feedback on your opinionated design and iterate. Best way is to find a possible user, give them the app link and observe what they do.

Try to record or take notes on the user interview.

When iterating, you’ll see that you’ll probably delete code and become even more opinionated.

Reuse code and you’ll see that an even simpler solution would have done a better job (even for the problem the reused code comes from).



# Particular for analytics tools
- be sure that it’s clear what controls are associated with what view.
- always offer a preview of what the data will be like or some cues that visualisations will be displayed if they do something with your buttons and sliders; otherwise users won't even try to operate your sliders since they assume there is nothing interesting to be seen there.
- don't underestimate the power of a good data grid