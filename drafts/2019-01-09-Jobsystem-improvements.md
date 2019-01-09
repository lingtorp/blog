JobSystem Improvements

Quick blog post about changes to the job system that resulted in a nice performance increase.

Too bad that I did not take real measurements but on the other hand the performance increases at this point is so large that they are either really obviously good or bad. It helps make some performance changes easier but later on I really need to gather hard numbers and judge things against them rather than the FPS meter.

Added try_peeq in order to avoid the scenario where the workers always locked a mutex and checked the value. Just wanted them to check if they could check the value without locking.

Before and after code. Add the commit ids with some nice links to the actual commits.

1. Added try_peeq to worker execute loop
    1. Very minor impact (-+ 0-5 FPS), I did give up here but tried some other places
2. Decreased the amount of time the workers slept
    1. Since they don’t need to lock in each loop iteration I figured the sleep could be lower. From 1 millisecond to 1 microsecond.
        1. Large impact (+ cirka 75 FPS), sweet! 
3. Submission to the jobsystem using try_peeq 
    1. Forgot to check exactly how this changed but seems to not have any effect
4. Jobsystem function wait_on_all now avoids blocking AND waiting by using try_peeq
    1. Large impact (+ cirka 100 FPS), wow!

All in all the example scene went from around 240 FPS (+- 20 FPS) to around 400 FPS (+- 20 FPS).
From the lower bound to the lower bound that’s an increase of 72%! I feel like this in a sense validates my suspicion that the major computational workload is updating the transforms.


