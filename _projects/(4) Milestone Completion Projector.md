---
name: Milestone Completion Projector
tools: [Python, numpy, matplotlib, Asana API]
image: "/assets/image/milestone_completion_projection.png"
description: Projects a completion date for an Asana milestone without requiring task estimates.
---

# Milestone Completion Projector

As a software engineer, making accurate estimations for tasks is difficult.
If you're unfamiliar with the code you're touching or you've never done a task like that before (which will often be the case), you may not have much confidence in the accuracy of your estimate.

To improve its accuracy, you'll have to dig around in the code to get familiar with it and do a little research on libraries to scope the amount of necessary work.
At that point, half the task is already done and you might as well do the other half -- the actual implementation.
Unfortunately, you'll probably have other tasks to estimate as well. So instead, you'll have to context-switch away and context-switch back later, burning valuable time and energy.

![computer_frustration]({{ site.baseurl }}/assets/image/computer_frustration.jpg)

## What if I told you...

There's a better way. Milestones are usually broken down into similarly-sized tasks.
If we just get a few days into a milestone, we can plot a trend line for the task completion rate and compare it against the total number of tasks.
Where the trend line crosses the total number of tasks, the milestone is projected to be completed.
This method turns out to be surprisingly accurate, even if tasks are added mid-milestone, which is practically a certainty.

Here's a plot of an actual completed milestone my team at Lumanu worked on.
For demonstration purposes, the trend line here ignores the last four days of data, and we see that it predicts the completion date accurately in this case.

![milestone_completion_projection]({{ site.baseurl }}/assets/image/milestone_completion_projection.png)

## Extrapolation (trend line) methods

I tried two different extrapolation methods for the task completion rate:

#1. Line of best fit<br>
#2. Line passing through (0, start date) and (# tasks completed, today)

For the task addition rate, in addition to the above, I tried a dead-simple third method:

#3. Horizontal line at y = total number of tasks.

It turns out the combination of #2 for the task completion rate and #3 for the task addition rate was the most accurate (hence the above graph).

To determine this, I simulated what each combination of methods would have predicted the completion date to be during each day in the milestone and compared the predictions against the actual completion date.
Then I made a sort of "error bar graph" of the results. And I repeated this for multiple milestones. Here's one for the best combination, for the same milestone as above:

![milestone_completion_projection_errors]({{ site.baseurl }}/assets/image/milestone_completion_projection_errors.png)

Early on in the milestone, the projections are not very accurate, but about halfway through, the errors become quite small.
This is pretty good considering the inconsistent task addition and completion rate throughout this milestone.

## Conclusion

Engineers tend to underestimate the time it takes to complete tasks, causing deadlines, which are based on those estimates, to slip often.
Instead, we can just look at the actual time they take to complete the tasks and project a completion date based on that.
This both saves engineers' time and is more accurate 💪.