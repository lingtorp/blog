---
layout: post
title: "Job system improvements"
last_modified_at: 2019-01-09
---
# Job system Improvements

This post will be about changes to the job system that resulted in some nice performance increases.

Too bad that I did not take real measurements but on the other hand the performance increases at this point is so large that they are either obviously really good or really bad. It helps make some performance changes easier but later on I really need to gather hard numbers and judge things against them rather than the FPS meter.

Full source code is available here: https://github.com/Entalpi/MeineKraft

# try_peeq
```cpp
/// Try to check if the value of the semaphore is equal to i or not (TRY_PEek_EQuals)
bool try_peeq(const size_t i) {
  std::unique_lock<std::mutex> lk(mut, std::try_to_lock);
  if (lk.owns_lock()) { return value == i; }
  return false;
}
```

I added the function try_peeq to my semaphore struct (see above) in order to avoid the scenario where the workers always locked a mutex and checked the value. Just wanted them to check if they could check the value without locking.

The main thing to not about the function is that it will return false even though the value was not checked. This is fine given that we usually only want to perform a certain operation when the value is guaranteed to be equals to the value we passed it. As a result we will need to check this more often but the idea was that the part that is costing us performance is the blocking of the mutex itself therefore I made sense to just try to lock it multiple times instead. 

Here is the blocking version that was used before for reference.

```cpp
/// PEek EQuals
bool peeq(const size_t i) {
  std::lock_guard<std::mutex> lk(mut);
  return value == i;
}
```

# Worker execute loop
I tried using the added try_peeq in the worker execute loop but the performance impact was minor (if not non-existent ...). I did not give up here but decided to try some other places in the job system.

## Before
```cpp
void execute() {
  while (!sem.peeq(3)) {
    if (sem.peeq(1)) {
      sem.try_wait();
      workload();
      sem.post(2);
    } else {
      std::this_thread::sleep_for(std::chrono::milliseconds(1));
    }
  }
}   
```
## After
```cpp
void execute() {
  while (!sem.try_peeq(3)) {
    if (sem.try_peeq(1)) {
      sem.try_wait();
      workload();
      sem.post(2);
    } else {
      std::this_thread::sleep_for(std::chrono::microseconds(1));
    }
  }
}
```
When looking this over I realized that waiting a whole millisecond might be too long. Decreasing this to 1 *microsecond* gave an increase of over **70 FPS**. 

# Work submission
The next place I looked at was the part that passes the workload to the workers. Previously I used the blocking version of the function peeq which was maybe not ideal. 
```cpp
// Submits the work to be perform to the first available worker
ID execute(const std::function<void()>& func) {
  uint64_t i = 0;
  while (true) {
    if (thread_pool[i].sem.try_peeq(2)) {
      thread_pool[i].workload = std::move(func);
      thread_pool[i].sem.try_wait();
      return i;
    }
    i = (i + 1) % thread_pool.size();
  }
}
```
Unfortunely I forgot to check the effect of this change in detail.

# wait_on_all
Now this is where things get interesting. The function in question is wait_on_all which blocks until the execution of all workers are done. Since the job system is only used from the main thread this is the place where we "join" all the workers. It gets called once per frame so this is used a lot. 

In this form the function simply goes through each and every worker and wait until the worker is done before proceeding to the next worker. This is both blocking, via the peeq function and blocking by sleeping the current calling thread (**yikes!**). 
```cpp
// Blocks the current thread by waiting for each worker to finish
void wait_on_all() {
  for (size_t i = 0; i < thread_pool.size(); i++) {
    while (thread_pool[i].sem.peeq(0)) {
      std::this_thread::sleep_for(std::chrono::microseconds(1));
    }
  }
}
```

```cpp
// Blocks the current thread by waiting for each worker to finish
void wait_on_all() {
  uint64_t done = 0; 
  while (done != thread_pool.size()) {
    done = 0;
    for (uint64_t i = 0; i < thread_pool.size(); i++) {
      if (thread_pool[i].sem.try_peeq(2)) {
        done++;
      }
    }
  }
}
```
Now the function goes through each and every worker and notes down how many are done with their work. If the count is not the total amount of worker then we simply ask all of them again. 

# Summary
All in all the example scene went from around 240 FPS (+- 20 FPS) to around 400 FPS (+- 20 FPS).
From the lower bound to the lower bound that’s an increase of **72%**! Thats when blog-based-debugging hit me hard. I noticed a bug in wait_on_all that made it just check ONE worker, **d'oh**. Fixing this bug made the whole thing just slightly slower at cirka **390 FPS** for the lower bound.

This made me investigate. Here is the, only place, where work submission takes place. This is what I looked like before.
```cpp
void Renderer::update_transforms() {
  std::vector<ID> job_ids(graphics_batches.size());
  const std::vector<ID> t_ids = TransformSystem::instance().get_dirty_transform_ids();
  
  // A NUMBER OF SMALLER JOBS
  for (size_t i = 0; i < graphics_batches.size(); i++) {
    ID job_id = JobSystem::instance().execute([=]() { 
      auto& batch = graphics_batches[i];
      for (const auto& t_id : t_ids) {
        // Do some work for each object (omitted for clarity)
      }
      std::memcpy(batch.gl_depth_model_buffer_object_ptr, batch.objects.transforms.data(), batch.objects.transforms.size() * sizeof(Mat4f));
      std::memcpy(batch.gl_bounding_volume_buffer_ptr, batch.objects.bounding_volumes.data(), batch.objects.bounding_volumes.size() * sizeof(BoundingVolume));
    });
    job_ids.push_back(job_id);
  }

  JobSystem::instance().wait_on_all();
}
```
What this function does is going over each of the batches and launching a job for it. This job in turn updates each transform for every object in that batch. In essence this workload is trivially parallelizable so it made sense at the time to just use one job per batch. Turns out this method causes so much overhead that it is actually lower than just running it all on one thread.

```cpp
void Renderer::update_transforms() {
  std::vector<ID> job_ids(graphics_batches.size());
  const std::vector<ID> t_ids = TransformSystem::instance().get_dirty_transform_ids();

  // ONE LARGE JOB
  ID job_id = JobSystem::instance().execute([=]() { 
    for (size_t i = 0; i < graphics_batches.size(); i++) {
        auto& batch = graphics_batches[i];
        for (const auto& t_id : t_ids) {
          // Do some work for each object (omitted for clarity)
        }
        std::memcpy(batch.gl_depth_model_buffer_object_ptr, batch.objects.transforms.data(), batch.objects.transforms.size() * sizeof(Mat4f));
        std::memcpy(batch.gl_bounding_volume_buffer_ptr, batch.objects.bounding_volumes.data(), batch.objects.bounding_volumes.size() * sizeof(BoundingVolume));
    }
  });
  job_ids.push_back(job_id);

  JobSystem::instance().wait_on_all();
}
```

Being a little bit curious at this point I wanted to check how the performance changes if I simply put all of the work in one single job. In theory I thought that, yes, the jobsystem overhead is large but is it larger than the computational cost? Turns out: no. The overhead decreased obviously and thus the performance increased to cirka **470 FPS**. 

```cpp
void Renderer::update_transforms() {
  std::vector<ID> job_ids(graphics_batches.size());
  const std::vector<ID> t_ids = TransformSystem::instance().get_dirty_transform_ids();

  // NO JOBS
  for (size_t i = 0; i < graphics_batches.size(); i++) {
      auto& batch = graphics_batches[i];
      for (const auto& t_id : t_ids) {
        // Do some work for each object (omitted for clarity)
      }
      std::memcpy(batch.gl_depth_model_buffer_object_ptr, batch.objects.transforms.data(), batch.objects.transforms.size() * sizeof(Mat4f));
      std::memcpy(batch.gl_bounding_volume_buffer_ptr, batch.objects.bounding_volumes.data(), batch.objects.bounding_volumes.size() * sizeof(BoundingVolume));
  }
}
```

At this point what is the point of having a job system that only seems to slow down the renderer? Good question, next one please. The overhead reduced to *zero* the performance jumped to, a staggering I might add, **560 FPS**. That is quite a jump from the *240 FPS* at the start. It is roughly an increase of 
**130%**. 