---
layout: post
title: You're up and running!
---


Many years ago I was working on a PABX call-centre monitoring system that was sold to customers of a large Australian Telco. It was written in 'c' (Using the Borland 4.51 C++ compiler on a 486) designed to run 24x7, monitoring a call centre and providing real-time stats on a Wall mounted display.

When I joined the company I had heard about an old bug that would occasionally occur (seemingly at random times) where the date field would become corrupted in their database records. As the system re-aligned its clock with the PABX every 15 minutes, only a few records would be impacted. It was not considered a high priority and as I had so little to go on and was busy with other issues, I didn't spend time on it. (On reflection I wonder if the PABX date alignment was done to cater for this issue?)

Eventually I was given a junior programmer to assist me to support/enhance the code base. I was explaining to him how the code worked and was running through a routine that was triggered when the clock struck midnight.
This code performed a roll-forward of the baseline date-time (which is held in UNIX style Seconds format).

Despite there being the 'time()' and 'mktime()' functions that handle this sort of thing, they used an odd routine that seemed to have been sourced elsewhere.

Something like the following logic occured when the system was started;
int CurrentMidnight = GetSecondsfromPABX();  # Set on program startup by asking the PABX

The routine executed at midnight looked something like this;

	RollMidnight()
	{
	...
		int dy, mth, yr;

		ConvertToDate(CurrentMidnight, dy, mth, yr);
		dy = dy + 1;
		CurrentMidnight = DateToSeconds(dy, mth, yr);
	...
	}

No comments in the code (no surprise here). It appears that the programmer wanted to roll the date forward so converted CurrentMidnight into DD/MM/YY. Added one to the day (DD) and converted it back via another function.

As I was talking my colleague through this section of code I immediately realised that I was looking directly at the offending code. This was the bug!

I told him,
"You've got half an hour. I want you to look at this code and tell me what's wrong, and how to fix it. I then want you to tell me why your solution is wrong".

As it also happened to be the last day of the month I went directly to the Level 1/2 support staff and asked them to contact all customers that ran this version of the code to let them know they need to shutdown the software just prior to midnight, and restart just after midnight. This would ensure that routine would not be used and no corruption would occur.

After half an hour I checked in with my colleague to see how he went.
Unfortunately he hadn't spotted the date-bug at all.

What I had assumed he would have seen is the highly dangerous;

	  dy = dy + 1;

This of course increments the date to the next day but has no bounds check.
Ie: If today is the 31st, then this code would make the date the 32nd.

Had he seen this I figured he would have had an elaborate solution to account for the last date in each month, then cater for leap years(last day of February would be 29th). That approach would have meant rolling the month forward and resetting the date to 1.
For end of year, 31/12, he might have reset the day/month to 1/1 and rolled the year forward.

This approach would have worked but I immediately dismissed it as the solution was really so much simpler and that was what I had meant when I wanted to him to tell me why his solution was wrong.

The actual requirement was;
Starting with a value in seconds, increment to a value in seconds that represents midnight the next day

Actual Solution
A "#define" already existed and was in use in various other locations in the code, (even if it wasn't, it is easy enough to create)

	  #define SECONDS_IN_A_DAY 60*60*24 # seconds-in-minute * minutes-in-hour * hours-in-day

I replaced the function calls with this;

	  CurrentMidnight += SECONDS_IN_A_DAY;

This is all that was required, short simple and bug fixed.


The actual bug worked like this;

Suppose today is the last day of May, the date would be incremented, then the function call would look like this;
CurrentMidnight = DateToSeconds(32, 5, 95);

The DateToSeconds() function at least checked things and if it detected an error it would pass back -1.
So the code was essentially doing this;
	CurrentMidnight = -1;

For fans of the Y2038 issue (http://en.wikipedia.org/wiki/Year_2038_problem) that represents 03:14:07 UTC on Tuesday, 19 January 2038
I had not heard of Y2038 at that time but '-1' represents the largest number you can hold in an 32-bit int which is the year 2038.

So records were stamped with 2038 for 15 minutes till the next PABX date-realignment.

Of course this highlights the other flaw in the logic,
If a routine can return an error value, CHECK FOR IT!!

Now some of you noticed that I used '96' for the year. Yes it was only using a 2 digit year. THAT can of worms I shall cover in another article.