# Java Memory Management: Complete Beginner to Senior Level

This guide assumes you know NOTHING about Java memory. We'll build from absolute basics.

---

## PART 1: THE MOST BASIC CONCEPTS

### What is Memory?

Think of your computer's memory (RAM) like a warehouse:
- You put things in it (objects)
- You retrieve things from it (use objects)
- You remove things you don't need anymore (garbage collection)
- **When you run out of space, you can't add more → OutOfMemoryError (OOM)**

Java memory is that warehouse.

### What is an Object?

In Java, you create objects like this:

```java
User user = new User("John", 25);
```

This creates a **new object** that takes up space in memory (warehouse). Every object takes space.

### The Problem: Objects Pile Up

```java
// This creates 1 million objects!
for (int i = 0; i < 1_000_000; i++) {
    User user = new User("User_" + i, 20 + i);
    // user is created
    // user takes space in memory
    // user goes out of scope (we're done with it)
    // BUT... does Java automatically remove it from memory?
}
```

**This is why we need Garbage Collection (GC).**

### What is Garbage Collection?

Garbage Collection is **automatic cleanup**. Java automatically looks at all objects and says:
- "Is this object still being used?"
- "If NO → delete it, free up memory"
- "If YES → keep it, we need it"

**Problem:** Garbage collection takes time. While GC is running, your application **PAUSES**. We'll talk about this later.

---

## PART 2: THE HEAP (Where Objects Live)

### Heap = Warehouse of Objects

```
Heap Memory (2 GB total)
│
├─ Young Generation (where NEW objects go)
│  └─ This is where we focus first
│
└─ Old Generation (where LONG-LIVED objects go)
   └─ We'll explain this later
```

Every object lives in the Heap. The Heap is divided into two main regions: Young and Old.

### Why Two Regions?

**Observation:** Most objects die young.

```
Typical Java application:
- User types something, a handler creates objects (String, ArrayList, etc)
- After 100ms, handler finishes, request is done
- Those objects become garbage
- They lived only 100ms

But some objects live forever:
- Database connection object: created once, used for whole application lifetime
- Cache object: created once, reused thousands of times
```

So Java separates them:
- **Young Generation:** For short-lived objects (die quickly)
- **Old Generation:** For long-lived objects (survive for a long time)

---

## PART 3: YOUNG GENERATION (The Most Important Part)

### What is Young Generation?

**Young Generation = a small area of memory where NEW objects are created.**

Default size: **256-512 MB** (you can change this)

### How Young Generation Works (Step-by-Step)

Let me show you the actual timeline:

```
TIME: 0 milliseconds
Your Java application starts
Young Gen status: EMPTY, ready for objects

TIME: 10 milliseconds
Your code runs: User user = new User("Alice", 25);
Young Gen: [User object, 50 bytes used] = 50 bytes
Status: Plenty of space

TIME: 20 milliseconds
More objects created
Young Gen: [User, List, String, Integer, ...] = 50 MB
Status: Still okay

TIME: 500 milliseconds
LOTS of objects created (many requests processed)
Young Gen: 512 MB FULL! ← This is the problem!
Status: NO MORE SPACE!

What happens now? → GARBAGE COLLECTION!
```

### Garbage Collection: What Actually Happens

When Young Gen fills up:

```
TIMELINE of Young Generation GC:

500ms:  Young Gen is 100% full
        All application threads PAUSE ← Stop working!
        
501ms:  GC thread wakes up
        GC scans all objects in Young Gen
        GC asks: "Which objects are still being used?"
        
        Examples:
        - User object in current request → STILL USED ✓
        - String from previous request → NOT USED ✗ (garbage)
        - List from 2 requests ago → NOT USED ✗ (garbage)

520ms:  GC deletes garbage objects
        Young Gen now has space for new objects
        GC finishes
        
521ms:  Application threads RESUME ← Start working again!

Result: Application was PAUSED from 500ms to 521ms = 21ms pause!
```

### The Critical Question: What Happens to Surviving Objects?

Some objects survive the garbage collection:

```
Before GC:
Young Gen: [garbage1, garbage2, ALIVE_USER, garbage3, ALIVE_CACHE, ...]

After GC:
Young Gen: [ALIVE_USER, ALIVE_CACHE, ...] (garbage deleted)
           (space is freed!)
```

But here's the key question: **Where do the ALIVE objects go after many GCs?**

Answer: **They get promoted to Old Generation**

```
Timeline of an object's life:

Created:   Young Gen (age 0)
After GC1: Young Gen (age 1)
After GC2: Young Gen (age 2)
After GC3: Young Gen (age 3)
After GC4: Young Gen (age 4)
After GC5: Old Gen (age 5+) ← Promoted because survived many GCs!
```

---

## PART 4: OLD GENERATION

### What is Old Generation?

**Old Generation = area for objects that survived many Young Gen GCs**

Default size: **1-2 GB** (larger than Young Gen)

### Why is Old Generation Important?

```
Scenario:
You have a cache of 100 million user profiles
Each profile is 1 KB
Total: 100 MB cache

This cache:
- Created once when app starts
- Survives every Young Gen GC
- Gets promoted to Old Gen

Now in Old Gen:
- Cache: 100 MB
- Other long-lived objects: 200 MB
- Total Old Gen used: 300 MB of 1000 MB

If you fetch 10 million more profiles → cache grows to 1 GB
Now Old Gen is 1.3 GB of 1000 MB → FULL!

When Old Gen fills:
- BIG GC pause happens (seconds!)
- This is BAD for user experience
```

---

## PART 5: REAL EXAMPLE - STRING CONCATENATION

This is a real problem that kills performance:

```java
// BAD CODE - Creates MANY objects
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i + ",";  // This creates a NEW String each time!
}

// What happens in memory:
// Iteration 1: result = "0,"           → 1 String object created
// Iteration 2: result = "0,1,"         → NEW String created, old "0," becomes garbage
// Iteration 3: result = "0,1,2,"       → NEW String created, old "0,1," becomes garbage
// ...
// Iteration 10000: total garbage = 10,000 String objects!

// Young Gen is PACKED with garbage!
// GC runs constantly because Young Gen keeps filling up!
// App pauses every 100ms for GC!
```

This is REAL. Here's the fix:

```java
// GOOD CODE - Creates ONE object
StringBuilder result = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    result.append(i).append(",");  // Reuses same StringBuilder
}
String finalResult = result.toString();  // Only create final String once

// Memory usage:
// Only 1 StringBuilder + 1 final String = 2 objects
// Young Gen barely affected!
// GC runs rarely!
```

**Performance difference:**
- Bad code: GC every 100ms, 20ms pause = 20% CPU wasted on GC
- Good code: GC every 10 seconds, 20ms pause = 0.2% CPU wasted on GC

**That's 100x difference in GC overhead!**

---

## PART 6: HOW TO UNDERSTAND YOUR MEMORY USAGE

### Concept: Allocation Rate

**Allocation Rate = How many MB per second your app creates**

```
Example 1: Web service handling requests
- 1000 requests per second
- Each request creates 100 KB of objects
- Allocation = 1000 × 100 KB = 100 MB/s

With Young Gen = 512 MB:
- Time to fill Young Gen = 512 MB / 100 MB/s = 5.12 seconds
- GC happens every 5 seconds!
- If each GC pause = 20ms, then app pauses every 5 seconds for 20ms

Example 2: Same service, but optimized code
- 1000 requests per second
- Each request creates 20 KB of objects (5x less!)
- Allocation = 1000 × 20 KB = 20 MB/s

With Young Gen = 512 MB:
- Time to fill Young Gen = 512 MB / 20 MB/s = 25.6 seconds
- GC happens every 25 seconds!
- Much better for users!
```

**Key insight:** Reducing allocation rate is MORE important than increasing heap size!

### How to Measure Your Allocation Rate

Enable GC logging:

```bash
java -Xlog:gc*:file=gc.log:time,uptime MyApp
```

This creates a file that shows:

```
[0.100s] GC Young Gen: 512M -> 100M (took 15ms)
[5.200s] GC Young Gen: 512M -> 100M (took 18ms)
[10.300s] GC Young Gen: 512M -> 100M (took 16ms)
```

**Calculate:**
- GC happened at: 0.1s, 5.2s, 10.3s = every ~5 seconds
- Objects deleted: 512M - 100M = 412M per GC
- Allocation rate = 412M / 5s = 82 MB/s

If you want to improve: **reduce allocation from 82 MB/s to 20 MB/s** (like the string concatenation example).

---

## PART 7: THE MOST COMMON PROBLEM - MEMORY LEAKS

### What is a Memory Leak?

**Memory Leak = objects that should be garbage but aren't**

```java
// MEMORY LEAK EXAMPLE
public class UserCache {
    private static Map<Long, User> cache = new HashMap<>();
    
    public User getUser(long id) {
        return cache.computeIfAbsent(id, userId -> {
            return User.fetch(userId);  // Fetch from DB
        });
    }
    // No way to remove users from cache!
}

// What happens:
// Request 1: getUser(1) → cache has {1: User1}
// Request 2: getUser(2) → cache has {1: User1, 2: User2}
// ...
// Request 1,000,000: getUser(1000000) → cache has 1,000,000 users!

// At 1 KB per user:
// cache = 1 GB!

// After a few hours: cache = 10 GB
// After a few days: Runs out of heap memory → OOM!
```

### How to Fix Memory Leaks

Add bounds and cleanup:

```java
// FIXED VERSION
public class UserCache {
    private static final Map<Long, User> cache = new HashMap<>();
    private static final int MAX_SIZE = 10_000;
    
    public User getUser(long id) {
        synchronized(cache) {
            // If cache is too big, clear it
            if (cache.size() >= MAX_SIZE) {
                cache.clear();
            }
            
            return cache.computeIfAbsent(id, userId -> {
                return User.fetch(userId);
            });
        }
    }
}

// Now:
// Cache max size = 10,000 users = ~10 MB
// Heap is safe!
```

---

## PART 8: UNDERSTANDING GC PAUSE TIME

### What Determines How Long GC Pauses?

GC pause time depends on:

```
1. How many objects are in Young Gen?
2. How many of those objects are still alive?
```

**Example:**

```
Scenario A: 512 MB Young Gen, 100 MB of alive objects
- GC must copy 100 MB from Young Gen to Old Gen
- On modern hardware: ~100 MB takes ~5-10 milliseconds
- Pause: 5-10ms ✓ (acceptable)

Scenario B: 512 MB Young Gen, 400 MB of alive objects (most objects survive!)
- GC must copy 400 MB from Young Gen to Old Gen  
- On modern hardware: ~400 MB takes ~40-80 milliseconds
- Pause: 40-80ms ✗ (problematic for latency-sensitive services)

Scenario C: Young Gen filled with massive objects (byte arrays, large strings)
- GC must copy hundreds of MB
- Pause: 100-500ms ✗✗ (very bad)
```

**Key insight:** If too many objects survive each Young Gen GC, the pause time gets longer!

---

## PART 9: INTERVIEW QUESTIONS & ANSWERS

### Q: "Explain Young Generation to me"

**Answer:**

"Young Generation is where all new objects are allocated. It's small (512 MB typically) so it fills up fast. When it fills up, Java pauses your application to run garbage collection - it scans all objects, deletes the garbage, keeps the alive ones, and then resumes.

The problem is: every GC pause stops your app, even if just for 20 milliseconds. If your app allocates 100 MB/second and Young Gen is 512 MB, you get a pause every 5 seconds.

Objects that survive many Young Gen GCs eventually get moved to Old Generation, which is larger but collects less frequently.

The key is: reduce allocation rate in your code, not increase heap size."

### Q: "How would you reduce GC pressure?"

**Answer:**

"Three things:

1. **Reduce allocation rate** - Don't create unnecessary objects
   - Instead of string concatenation (`str += x`), use StringBuilder
   - Instead of creating Lists per request, reuse them
   - Cache objects instead of recreating them

2. **Reduce live set** - Don't keep objects longer than needed
   - Add TTL to caches (remove old entries)
   - Add size limits to collections
   - Clean up references when done

3. **Choose right GC collector** - If above doesn't work, tune GC
   - G1GC for balanced latency/throughput
   - ZGC for ultra-low pause times"

### Q: "What causes OutOfMemoryError?"

**Answer:**

"Multiple reasons, not just heap:

1. **Heap OOM** - Too many objects, can't fit in heap
   - Fix: reduce allocation, add cache bounds, fix leaks

2. **Memory Leak OOM** - Objects that should be garbage aren't
   - Fix: find what's holding references, add cleanup code

3. **Old Generation OOM** - Too many objects survived to old gen
   - Fix: reduce promotion rate (keep objects shorter-lived)

4. **Metaspace OOM** - Too many classes loaded
   - Fix: reduce dynamic class generation"

---

## PART 10: PRACTICAL CHECKLIST

✓ **Understand**: Young Gen is where NEW objects go
✓ **Understand**: Young Gen fills up, GC runs, app pauses
✓ **Understand**: Reduce allocation rate = fewer GCs
✓ **Understand**: Memory leaks = objects not garbage when should be
✓ **Fix leaks**: Add bounds and cleanup to caches
✓ **Reduce allocation**: Use StringBuilder instead of string +
✓ **Reduce allocation**: Reuse objects instead of creating new ones
✓ **Measure**: Enable GC logs to see what's happening
✓ **Analyze**: Calculate allocation rate from GC logs

---

---

## PART 11: DEEP DIVE - GARBAGE COLLECTION ALGORITHMS

### The Mark-Sweep-Compact Algorithm (Baseline)

This is the foundation all modern GCs are built on. Here's exactly how it works:

```
STEP 1: MARK PHASE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Heap before marking:
┌─────────────────────────────────────────────────────────────┐
│ [User1] [List1] [String1] [Cache1] [User2] [abandoned] ... │
│                                             ↑ garbage
└─────────────────────────────────────────────────────────────┘

Step 1: GC walks through all LIVE REFERENCES
- Finds User1 → mark it as ALIVE
- User1 references List1 → mark List1 as ALIVE
- List1 contains String1 → mark String1 as ALIVE
- User1 references Cache1 → mark Cache1 as ALIVE
- User2 is NOT referenced by anything → NOT marked
- [abandoned] is NOT referenced → NOT marked

Heap after marking:
┌─────────────────────────────────────────────────────────────┐
│ [User1✓] [List1✓] [String1✓] [Cache1✓] [User2✗] [X✗] ... │
│          marked   marked      marked             garbage
└─────────────────────────────────────────────────────────────┘

This process scans EVERY SINGLE OBJECT and follows EVERY reference.
Time complexity: O(heap_size)
```

```
STEP 2: SWEEP PHASE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

GC iterates through heap from start to end:
- User1✓ → KEEP
- List1✓ → KEEP
- String1✓ → KEEP
- Cache1✓ → KEEP
- User2✗ → DELETE (return memory to free list)
- [abandoned]✗ → DELETE (return memory to free list)

Heap after sweep:
┌─────────────────────────────────────────────────────────────┐
│ [User1] [List1] [String1] [Cache1] [FREE SPACE] [FREE] ... │
│                                     ↑ can use this
└─────────────────────────────────────────────────────────────┘

Problem: Memory is fragmented!
Free space is scattered, not contiguous.
```

```
STEP 3: COMPACT PHASE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

GC moves all ALIVE objects to one side:

Before compaction (fragmented):
┌─────────────────────────────────────────────────────────────┐
│ [User1] [List1] [String1] [Cache1] [FREE] [FREE] [FREE] ... │
└─────────────────────────────────────────────────────────────┘

After compaction (contiguous):
┌─────────────────────────────────────────────────────────────┐
│ [User1] [List1] [String1] [Cache1] [FREE SPACE..........] │
│                                     ↑ large contiguous block
└─────────────────────────────────────────────────────────────┘

Benefits:
- NEW objects can be allocated quickly (just use next position)
- Cache-friendly (objects are packed together)

Drawback:
- Compaction requires copying objects = expensive!
- All references must be updated to new addresses
```

### Copying Algorithm (Young Generation)

Young Generation uses a clever variation called **Copying GC**:

```
Why copy instead of mark-sweep-compact?

Young Gen assumption: Most objects die immediately
┌────────────────────────────────────────┐
│ Objects: [garbage] [garbage] [ALIVE]   │
│          95% garbage, 5% alive         │
└────────────────────────────────────────┘

Mark-sweep-compact would:
1. Mark (expensive - examine 100 objects)
2. Sweep (expensive - delete 95 objects)
3. Compact (expensive - move 5 objects)
Total: expensive for little benefit

Copying GC instead:
1. Copy only the 5 ALIVE objects
2. Ignore the 95 garbage (they disappear automatically)
3. Done!
```

**Copying GC: Step by Step**

```
Young Generation memory layout:
┌─────────────────────────────────────────┐
│         Active (A)      │    Inactive (B)│
│ [obj1] [obj2] [obj3]    │ [empty]       │
└─────────────────────────────────────────┘

When Young Gen (A) is full:

Step 1: Root scanning
- Find all objects referenced from outside Young Gen
- Example: [obj1] and [obj3] are referenced
         [obj2] is not referenced

Step 2: Copy alive objects to B
┌─────────────────────────────────────────┐
│ [empty]                 │ [obj1] [obj3]  │
└─────────────────────────────────────────┘

Step 3: Update all references
- obj1 old location (0x1000) → new location (0x5000)
- obj3 old location (0x2000) → new location (0x5500)

Step 4: Flip spaces
- Now B becomes active
- Old A becomes inactive (ready for next GC)
- obj2 (garbage) is automatically discarded

Result: Young Gen is now empty again!
Pause time: 10-30ms (fast because only copy alive objects)
```

### Generational Hypothesis

Why separate Young and Old generation?

```
OBSERVATION: Age Distribution of Objects

In a typical Java application:
- 90% of objects die within their first allocation
- 10% survive the first GC
- Of those 10%, only 2% survive the second GC
- Of those 2%, only 0.5% survive the third GC
- ...

Graph:
Survival rate (%)
│
100├─ Generation 0 (just created)
  │ ╲
 80├  ╲
  │   ╲
 60├    ╲
  │     ╲
 40├      ╲
  │       ╲
 20├        ╲═════════════════════ Old Gen (stable)
  │ 
  0└──────────────────────────────────────
    0    1    2    3    4    5    GC #
```

**Implication:**
- Young Gen: GC very frequently, very fast (copy only 5-10% survivors)
- Old Gen: GC rarely, takes longer (mark-sweep-compact many objects)

### Write Barriers (The Clever Part)

Here's a tricky question: **What if an Old Gen object references a Young Gen object?**

```
Memory layout:
┌─────────────────────────────┐
│ Old Gen: [OldObj1]          │  OldObj1 → YoongObj1
│                             │  (cross-generation reference)
├─────────────────────────────┤
│ Young Gen: [YoungObj1]      │
└─────────────────────────────┘

When Young Gen GC runs:
- GC scans Young Gen
- Finds YoungObj1 is referenced from OldObj1
- Must keep YoungObj1 alive!

But problem: GC doesn't want to scan entire Old Gen
(that would be slow and defeat the purpose of generational GC)

Solution: WRITE BARRIER
```

**Write Barrier: How It Works**

```java
// When you do this:
oldObject.field = youngObject;  // Old → Young reference

// The JVM automatically executes:
if (oldObject.age > youngObject.age) {
    // This is a cross-generation reference!
    // Add it to "remembered set" so GC knows about it
    REMEMBERED_SET.add(oldObject);
}

// During Young Gen GC:
for (Object root : ROOTS) {
    scan(root);  // Normal scanning
}
for (Object remembered : REMEMBERED_SET) {
    scan(remembered);  // Also scan old objects that reference young objects
}
```

**Cost:**
- Small overhead when assigning to fields
- Big benefit: doesn't need to scan entire Old Gen

### Tri-Generational (Deprecated but Educational)

Java used to have THREE generations:

```
Young Gen (0-512 MB):
  ├─ Eden (new allocations)
  └─ Survivor (objects that survived one GC)

Old Gen (1-2 GB):
  └─ Long-lived objects

Permanent Gen (64-256 MB):  ← Deprecated in Java 8
  └─ Class metadata, constant pool, internalized strings
  └─ Replaced with Metaspace (unlimited)
```

---

## PART 12: MODERN GC ALGORITHMS

### G1GC (Garbage First) - The Most Popular

G1GC is the **default GC since Java 9**. It replaces the generational model with something radically different:

```
Key Idea: Divide heap into REGIONS instead of Young/Old

Traditional:
┌──────────────────────────────────────────┐
│ Young Gen (small, frequent GC)           │
│                                          │
│                                          │
│ Old Gen (large, infrequent GC)          │
│                                          │
│                                          │
└──────────────────────────────────────────┘

G1GC:
┌──────────────────────────────────────────┐
│ [R1]Y [R2]Y [R3]O [R4]O [R5]Y [R6]O ... │
│ [R7]O [R8]Y [R9]O [R10]Y [R11]E [R12]O │
│ ... (regions are 1-32 MB each)          │
└──────────────────────────────────────────┘

Y = Young region
O = Old region  
E = Eden region
```

**Why Regions?**

```
Problem with traditional GC:
- To GC Old Gen, must scan entire Old Gen (slow)
- Pause time: 1-10 seconds! (unacceptable)

G1GC solution:
- Track which regions have dead objects
- GC only the regions with most garbage
- Skip regions with mostly live objects
- Pause time: ~200ms (much better)

Example:
Old Gen has 10 regions, each 100 MB:
- Region 1: 95% garbage, 5% alive
- Region 2: 95% garbage, 5% alive
- Region 3: 30% garbage, 70% alive
- Region 4: 30% garbage, 70% alive
- ... (more regions)

G1 says: "GC regions 1 and 2 first (most garbage)"
- Reclaim 190 MB in ~100ms
- Skip region 3 and 4 (not worth the pause)
```

### ZGC - Ultra-Low Latency

For applications that need **sub-10ms pause times**:

```
Traditional GC timeline:
[App running] [App pauses 200ms for GC] [App running]
              ↑ unacceptable for latency-sensitive apps

ZGC timeline:
[App running] [pause 1ms] [pause 1ms] [pause 1ms] [App running]
              ↑ concurrent work doesn't pause app
```

**How ZGC achieves <10ms pauses:**

```
Key innovation: Concurrent marking and compaction

Traditional concurrent GC:
1. Mark (concurrent) - may happen while app runs
2. Compact (paused) - must pause (updating references)
3. App resumes

ZGC:
1. Mark (concurrent) - app keeps running
2. Update references (concurrent) - app keeps running
3. Compact (concurrent) - app keeps running
4. Brief pause (1-10ms) - only for critical sections

Cost: Uses 30% more memory and CPU
Benefit: Pause times < 10ms (industry-changing)
```

### Shenandoah GC - RedHat's Low-Pause Alternative

Similar to ZGC but:
- Developed by RedHat
- Fewer memory overhead than ZGC
- Supports older Java versions

---

## PART 13: HEAP ANALYSIS - REAL EXAMPLE

### Analyzing OOM: Finding Memory Leaks

```
Scenario: Your application runs fine for 1 hour, then crashes with OOM

Symptoms:
java.lang.OutOfMemoryError: Java heap space
at java.util.Arrays.copyOf()

Question: Where did the extra memory go?
Answer: Use a heap dump
```

**Taking a Heap Dump:**

```bash
# Method 1: Automatic on OOM
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof MyApp

# Method 2: While app is running
jmap -dump:live,format=b,file=heapdump.hprof <PID>

# Method 3: Using jcmd (Java 9+)
jcmd <PID> GC.heap_dump /tmp/heapdump.hprof
```

**Analyzing the Dump with jhat:**

```bash
jhat heapdump.hprof
# Opens web interface at http://localhost:7000
```

**What to Look For:**

```
1. Find largest objects
   - Click "Show heap histogram by class"
   - Sort by size
   
   Example output:
   Class Name            Instances  Total Size
   byte[]                50,000     1.5 GB      ← Too much?
   String                100,000    500 MB      ← Too much?
   HashMap               5,000      100 MB      ← Reasonable
   
2. Investigate the culprit
   - Click byte[] class
   - Find which object holds the reference
   - Track back to root
   
3. Example chain:
   byte[] (1.5 GB)
     ← held by ByteBuffer
       ← held by FileInputStream
         ← held by FileCache
           ← held by static field in CacheManager
           
   Finding: FileCache is never closed! (memory leak)
   
4. Fix:
   public class FileCache {
       public void close() {
           inputStream.close();  // Release memory
       }
   }
```

---

## PART 14: TUNING JVM FLAGS

### Most Important Flags

```bash
# Set heap size
-Xms2G          # Initial heap size = 2 GB
-Xmx2G          # Max heap size = 2 GB

# Set Young Gen size
-Xmn512M        # Young Gen = 512 MB

# Choose GC algorithm
-XX:+UseG1GC              # Use G1GC (default Java 9+)
-XX:+UseZGC               # Use ZGC (Java 11+)
-XX:+UseParallelGC        # Use Parallel GC (good throughput)
-XX:+UseConcMarkSweepGC   # Use CMS (low latency, deprecated)

# G1GC specific tuning
-XX:MaxGCPauseMillis=200  # Target 200ms pauses

# Enable GC logging
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# Aggressive optimization
-XX:+AggressiveOptimization
-XX:+UseFastAccessorMethods
```

### Real Tuning Example

**Scenario:** Your application has 50ms p99 latency target, but GC pauses are 500ms

```bash
# Bad config (old JVM):
java -Xmx4G MyApp
# Uses default CMS GC with 4GB heap
# Pause times: 500-2000ms ✗

# Good config (modern JVM):
java -Xms2G -Xmx2G -Xmn512M -XX:+UseG1GC -XX:MaxGCPauseMillis=50 MyApp
# Uses G1GC, targets 50ms pauses
# Pause times: 30-80ms ✓

# Better config (ultra-low latency):
java -Xms4G -Xmx4G -XX:+UseZGC -Xlog:gc*:file=gc.log MyApp
# Uses ZGC, pause times < 10ms
# Memory overhead: ~30% more
# CPU overhead: ~5% more
# But achieves latency target ✓
```

---

## PART 15: GC ALGORITHM COMPARISON TABLE

| Algorithm | Pause Time | Throughput | Memory Usage | Java Version | Best For |
|-----------|-----------|-----------|--------------|--------------|----------|
| **Serial GC** | 100-1000ms | 95% | Baseline | 1.0+ | Single-core, small heaps |
| **Parallel GC** | 50-500ms | 99% | +10% | 1.4+ | Batch processing, throughput |
| **CMS** | 50-200ms | 95% | +30% | 1.4-14 | Web services (deprecated) |
| **G1GC** | 50-200ms | 98% | +5% | 7+ | Balanced, large heaps |
| **ZGC** | <10ms | 97% | +30% | 11+ | Ultra-low latency |
| **Shenandoah** | <10ms | 97% | +20% | 12+ | Low pause alternative |

---

## FINAL THOUGHT

**You now understand Young Generation.**

You know:
- Why it exists
- What happens when it fills up
- How to reduce GC pressure
- How to find and fix memory leaks
- Mark-Sweep-Compact algorithm details
- Copying GC and write barriers
- Modern algorithms (G1GC, ZGC)
- How to analyze heap dumps
- How to tune JVM for your use case

That's **expert-level knowledge**. Most senior developers struggle with this depth.

Practice explaining:
1. "Why do we need two generations?"
2. "How does G1GC differ from Parallel GC?"
3. "Walk me through a GC pause timeline"
4. "How would you diagnose an OutOfMemoryError?"

That's how you'll nail the interview.
