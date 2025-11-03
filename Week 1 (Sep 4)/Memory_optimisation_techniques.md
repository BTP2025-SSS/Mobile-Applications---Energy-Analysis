1. Activities
- It is a view or single screen the users sees, login screen, dashboard screen etc.
- They are created and destryed on demand.

2. Framgments
- They are parts of the views. (resuable parts of ui)
- Created when the activity is created

### GARBAGE COLLECTOR
- The GC deletes only objects that are unreachable — i.e., there are no active references to them from any chain of GC roots (like static fields, active threads, JNI references, etc.).

### Context
- A context is a handle to the whole subsystem of android os.
1. Access to resources, themes, layouts.
2. References to system services (WindowManager, PowerManager)
3. Start activities (startActivity())
4. Access databases, SharedPreferences, layouts, etc
#### Activity Context
- Used using **this** and lives in the scope of the activity.
#### Application Context
- Lives as long as the app process.
- Global for whole app, and ahs no references to the ui/system resources.

## View
- It contais refrence to the parent view, Context and some data that is being used.

# Memeory optimisation techniques

- In general all android app use garbage collectors.
- When java objects are created but no freed , then gc takes them and frees them, 
- When a large aboject needs space, the gc frees the dead objects to alllocate space to the new onr, this causes delays

## Memeory Leak causes

### 1. Static References to Views or Contexts
- When we reference to a static, they live as long the app lives, so if a activity if referenced by some static field, then even after deleting the activity, the static filed stays.
- Gc wont delete it because , gc only frees the memory when nothing is referencin the activity, as we have used static even after completing the activity the static filed refernces to it.

- And we can use satic data because they are not linked with the context or views.
- But views and contexts have refernces to many things in ui and os. so keeing it as static will cause memory leaks after deletion.
### Fix
- We can either nullify the static variable while destroying it.
- Use WeakRefernces for the static fileds like context etc , if we really need static to these context/ activity
- Use application context inside the static classes.
- To detect we can use **LeakCanary**

### 2. Anonymous / Non-static Inner Classes
- In Java, every non-static inner class (including anonymous ones) automatically holds a reference to its outer class.
```java
class MyActivity extends Activity {
    void startTask() {
        new Thread(new Runnable() {
            //
        }).start();
    }
}
```
- The anonymous Runnable is a non-static inner class.
- Here the runnable object when the thread starts executing, will have a refernce to the MyActivity.
- So when the Activity is deleted the gc wont free it, as it children are still referncing to it.

### Fix
- Make the runnable static and if we need reference use weakRefenece
```java
class MyActivity extends Activity {

    void startTask() {
        new Thread(new MyRunnable(this)).start();
    }

    static class MyRunnable implements Runnable {
        private final WeakReference<MyActivity> activityRef;

        MyRunnable(MyActivity activity) {
            activityRef = new WeakReference<>(activity);
        }
s
        @Override
        public void run() {
            MyActivity activity = activityRef.get();
            if (activity != null) {
                // do background work safely
            }
        }
    }
}

```

### 3.Registered Listeners / Callbacks Not Unregistered
- If you register something (like a BroadcastReceiver, OnClickListener, or SensorListener), and forget to unregister it when Activity is destroyed, the system still holds your reference — keeping Activity in memory.
### Fix
- Always unregister these listners
```java
protected void onPause() {
    super.onPause();
    unregisterReceiver(myReceiver);
}
```

### 4. Cursors Not Closed
- While quering the database, Cursor c = db.query(...);
- If we did not call c.close it keeps all the file descriptors, refernces to data->memory leak.
### Fix
- Always close, or use try block.


### Bitmaps
- Bitmaps are used whenever you want to draw images on the screen, manipulate images, or perform graphics operations.
- A Bitmap is essentially a map of pixels — it represents an image in memory.
- Their size os too large.
### Fix
- Use Glide, Picasso, and Coil automatically manage these bitmaps.
- Use cache to store them in disk and use when needed.


## Managing GPU Buffers
- On desktops, it contains VRAM to store texture data, frame buffers etc., main meoery isnt effected.
- But in mobiles gpu share system memeory no dedicated VRAM, so consumes a alot of memeory in mobiles.

1. Buffer Caching
- When a app is in background , for faster relaunch the buffers are kept in memeory, so if the gpu buufers are large like some textures may lead to 100's of mb , may get memeory limit error.
2. Compression Tradeoff
- Compress GPU buffers when app is backgrounded to save RAM.
- this reduses memeory but costs cpu, battery, latency
### Fix
1. Use optimized image formats ( like Webp instead of PNG/JPEG).
2. Downscale bitmaps/textures to display size.
3. Use caching libraries that respect lifecycle


## Memeory Duplication
- Memory duplication occurs when multiple processes store identical data in separate pages of RAM.
### Fix
1. **KSM — Kernel Same-Page Merging**
- Merge identical pages across processes into a single read-only page, and use Copy-on-Write (CoW) if a process modifies it.

2. **zRAM — Compressed RAM Swap**
- When your device runs low on RAM, normally Linux would swap pages to disk (flash storage) to free RAM.
- Here Pages are compressed, Stored in a special zRAM block device in RAM.
- When needed it is decompressed.

3.**Memscope / Selective Deduplication**
- Instead of scanning eveything like in KSM to find duplicates, it focus on where we will find more duplicates.

## Adaptive Background App Management
1. The Zygote Process
- When you launch an app, Android doesn’t start a process from scratch, Instead, it forks the Zygote process.
- Zygote already has core system classes and frameworks loaded in memory.
- Forking is fast and efficient, reducing app launch time.
- It also reduces duplication.

2. LMK(low memeory killer)
- Runs in the kernel, Kills processes when free memory drops below thresholds. Uses priority levels / importance hierarchy.

3. OOMK (Out-Of-Memory Killer)
- Kernel-level last-resort mechanism, Triggers if memory is critically low.
- Similar process ranking, but more aggressive to prevent system crashes.

- These both LMK and OOMK will ignore the app launch cost and user experience, may kill a process which is more frequent and heavy, which in turn when called again causes lag.

4. SmartLMK Idea
- Improve traditional LMK/OOMK by considering app launch/usage stats.
- Collect usage statistics:
- Launch frequency, Launch time / resource cost.
- And assign penalty scores to apps:


## Micro-optimizations:

- Small code changes that efefct greatly in low memory devices like mobiles.
1. Avoid unnecessary allocations
```java
// Bad: allocating inside loop
for (int i = 0; i < n; i++) {
    List<String> list = new ArrayList<>();
    ...
}
// Better: reuse collection
List<String> list = new ArrayList<>();
for (int i = 0; i < n; i++) {
    list.clear();
    ...
}
```
2. Reuse the collections instead creating new ones.
- use ArrayList.clear() or StringBuilder.setLength(0) instead of creating.

3. String concatenation
```java
// Bad
String s = "";
for(String part : parts) {
    s += part;
}

// Better
StringBuilder sb = new StringBuilder();
for(String part : parts) {
    sb.append(part);
}
String s = sb.toString();
```

4. Avoid referenceing to objects that u no longer need.

### Static Analysis
- Tools 
1. Lint (checks memory leaks, unused resources, and performance issues.)
2. FindBugs / SpotBugs: Flags potential bugs, null pointer risks, or inefficient code.
3. PMD: Checks for code style violations, unused variables, or inefficient constructs.
- Limitations - false positives/negatives; rely on human judgement, static tools cannot always see runtime usage patterns.
- we can use this to find some obvious bugs and then use android profiler to find hot paths where we get performance issues.

## Partitioning memory / Software aging / Exerciser Monkey testing
- eat memory as separate virtual nodes or partitions instead of one global pool.
- Reliable node: System apps, essential services, Unreliable node: Third-party apps, games.
- Limit damage when one app misbehaves.
- pros - reduces sys wide crashes.
- cons - Strict OS-level isolation requires effort, memeory fragmentation.

### Software Aging
- long-running apps can gradually degrade over time due to, memory leaks, unerlased resources etc.
- Stress tests: Run app continuously under typical workload.
- Fuzzing / Monkey testing: Random UI inputs or sequences to exercise all code paths and find apps that consume all meory as time passes.
- Partition main memory into virtual nodes (reliable vs unreliable apps) so one node failure doesn’t kill everything; test for software aging with fuzzing (Monkey) and stress tests to detect memory growth over time.

# Replace Android GC with ARC (Automatic Reference Countingz)
- In ARC every object carries a refernce count. on each assignment of refernce its refernce count inc and on deleting it dec.
Cons - Cyclic references: If two objects reference each other strongly (A→B and B→A), their count neer drops to 0.
- In ios dev will mark certain count that shoudnt inc retain count, Breaks cycles automatically when no strong references remain.
- in Java Cycles are collected as long as they’re unreachable from root objects.
- ARC in android is run on GC, as java accepts gc semantics.

- Strong reference: Keeps object alive.
- Weak/unowned reference: Does not increase reference count. Breaks cycles.
- Automatic memory release: When the last strong reference is gone, ARC calls deinit and frees memory immediately.

- Changing into ARC requires atmic operation in ref count, safety, and every pointer needs ref count 
- It is pratically immpossible to convert totally to ARC.

- Here ARC as it uses ref count deltes objects when count is 0, unlike in GC it uses mark and sweep, it starts form root and mark its childern if it is alive or not, and scan the heap and free the dead.
- memeoy freeing happens during gc cycles.
