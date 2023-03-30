<div align=center>
    <img src="https://user-images.githubusercontent.com/36140389/228953999-995b7818-baee-465b-a152-6b7e46ac4399.png">
</div>

## Synopsis

Foliocopia conglomerates proximate burdened segments to forge a "self-contained territory."
Peruse [REGION_LOGIC.md](REGION_LOGIC.md) for intricate minutiae concerning the modus operandi through which Foliocopia
assembles adjacent partitions.
Each autonomous zone possesses an individualized chronometer cycle, oscillating at the
customary Minecraft temporal frequency (20TPS). The chronometer cycles are concurrently effectuated
utilizing a multithreaded pool. The prevailing thread has been rendered obsolete, as
each territory intrinsically encompasses its unique "principal thread" that carries out
the entire chronometer cycle.

For a server laden with dispersed participants, Foliocopia will generate numerous
scattered territories and concurrently oscillate them all employing an adjustable multithreaded
pool. Consequently, Foliocopia should exhibit optimal scalability for servers of this nature.

Foliocopia is a distinct venture in its own right, and it shall not be amalgamated into Paper
in any foreseeable temporal frame.

A preliminary yet constructive overview is indicated here: [Project Overview](https://rootbeer.co/foliocopia/overview).

## Inquisitorial Compendium

## Which server classifications can reap advantages from Foliocopia?
Server categories that inherently disperse participants across the landscape,
such as skyblock or SMP, will predominantly derive maximal benefits from Foliocopia. It is also advantageous for the server
to accommodate a substantial player enumeration.

## Upon which hardware shall Foliocopia optimally function?
The ideal scenario entails a minimum of 16 processing cores (excluding threads).

## What is the optimal methodology for configuring Foliocopia?
Initially, it is advisable to pre-generate the world, significantly diminishing the
necessity for chunk system worker threads.

## The subsequent enumeration is a remarkably approximate estimation derived from the trials
conducted prior to Foliocopia's test server deployment, accommodating approximately 330 players at its zenith. As such, it is not precise and warrants additional refinement –
consider it a foundational reference point.

The total core enumeration of the apparatus should be
evaluated. Subsequently, apportion threads for:

Netty IO: ~4 per 200-300 players
Chunk system IO threads: ~3 per 200-300 players
Chunk system workers if pre-generated, ~2 per 200-300 players
No optimal conjecture for chunk system workers exists if not pre-generated, as
on the test server, 16 threads were designated, yet chunk generation remained
sluggish at ~300 players.
GC Settings: Ambiguous. However, GC settings do allocate concurrent threads, necessitating
accurate knowledge of their quantity. This is typically achieved via the -XX:ConcGCThreads=n flag. Refrain from
confusing this flag with -XX:ParallelGCThreads=n, as parallel GC threads only operate when
the application is paused by GC and should not be considered in the calculation.
Upon completing the aforementioned allocations, the residual cores on the system, up to 80%
allocation (total threads allocated < 80% of available CPUs), can be
apportioned to tickthreads (under global configuration, threaded-regions.threads).

Refraining from allocating over 80% of the cores is crucial due to the
probability that plugins or even the server might utilize supplementary threads
whose configuration or prediction is unattainable.

Moreover, the aforementioned estimations are fundamentally rough approximations contingent on player count, rendering it highly plausible that thread allocation will be suboptimal. Consequently, you
will necessitate fine-tuning based on the observed utilization of the threads ultimately allocated.

## Plugin Conformity

The main thread is rendered obsolete. It is anticipated that every extant plugin
will mandate some degree of modification to function
cohesively with Foliocopia. Furthermore, the incorporation of any multithreading engenders
potential race conditions in plugin-held data, necessitating alterations.

Consequently, maintain compatibility expectations at nil.

## API Stratagems

At present, a plethora of API relies on the main thread.
It is projected that virtually no plugins compatible with Paper will
align with Foliocopia. Nonetheless, plans exist to incorporate API that
would enable Foliocopia plugins to synchronize with Paper.

For instance, the Bukkit Scheduler. The Bukkit Scheduler intrinsically
depends on a singular main thread. Foliocopia's RegionScheduler and Foliocopia's
EntityScheduler facilitate task scheduling for the "next tick" of whichever
region "possesses" a location or an entity. These could be implemented
in standard Paper, except they schedule to the main thread. In both instances,
task execution transpires on the thread that "owns" the
location or entity. This concept applies broadly, as the extant Paper
(single-threaded) can be perceived as an all-encompassing "region" that incorporates
all chunks in all realms.

The decision to add this API to Paper itself directly
or to PaperLib remains undetermined.

It is not yet decided whether to add this API to Paper itself directly
or to PaperLib.

### The Novel Guidelines

Initially, Foliocopia disrupts numerous plugins. To assist users in identifying which
plugins function, only plugins explicitly marked by the
author(s) as Foliocopia-compatible will be loaded. By inserting
`folia-supported: true` into the plugin's plugin.yml, plugin authors
can designate their plugin as congruous with regionized multithreading.

Another crucial rule is that regions tick in parallel, not
concurrently. They neither share nor expect to share data,
and data sharing will result in data corruption.
Code executing in one region must never
access or modify data present in another region. Simply
because multithreading is incorporated does not imply that everything
is now thread-safe. In reality, only a few elements were
rendered thread-safe to facilitate this process. As time progresses, the quantity
of thread context checks will only increase, even if it incurs a
performance penalty. No one will utilize or develop for a
server platform plagued with bugs, and the sole method to
prevent and identify these bugs is to ensure erroneous accesses fail hard at the
source of the access.

This necessitates Foliocopia-compatible plugins leveraging
API such as the RegionScheduler and the EntityScheduler to guarantee
their code operates within the appropriate thread context.

In general, it is prudent to assume that a region owns chunk data
approximately 8 chunks from the event source (i.e., player
breaks block, can likely access 8 chunks surrounding that block). However,
this is not guaranteed – plugins should capitalize on forthcoming
thread-check API to ensure correct behavior.

Thread-safety guarantees stem from the fact that a
single region owns data in specific chunks – and if that region is
ticking, it has unencumbered access to that data. This data is
exclusively entity/chunk/poi data and is entirely unrelated
to ANY plugin data.

Normal multithreading rules apply to data that plugins store/access
their data or another plugin's – events/commands/etc. are executed
in parallel because regions are ticking in parallel (synchronous execution is unattainable, as this gives rise to deadlock issues
and would undermine performance).

### Present API Augmentations

For a comprehensive understanding of API augmentations, kindly peruse
PROJECT_DESCRIPTION.md.

RegionScheduler, AsyncScheduler, GlobalRegionScheduler, and EntityScheduler
supplant the BukkitScheduler.
The entity scheduler can be obtained via Entity#getScheduler, while
the remaining schedulers are retrievable from the Bukkit/Server classes.
Bukkit#isOwnedByCurrentRegion examines if the presently ticking region
possesses specific positions/entities.

## Thread Milieus for API
For an in-depth comprehension of API augmentations, please consult
[PROJECT_DESCRIPTION.md](PROJECT_DESCRIPTION.md).

General heuristics:

1. Commands concerning entities/players are invoked within the region that owns
the entity/player. Console commands transpire in the global region.

2. Events entailing a solitary entity (e.g., a player breaking/placing a block) are
initiated within the region owning the entity. Events incorporating actions on an entity
(such as entity damage) are triggered within the region possessing the target entity.

3. The async modifier for events is deprecated – all events
originating from regions or the global region are deemed synchronous,
notwithstanding the absence of a main thread.

## Presently Dysfunctional API
- A majority of API that interacts with portals, respawning players, and some
player login API is impaired.
- The entirety of the scoreboard API is deemed non-functional (the global state has yet
to be properly implemented).
- World loading/unloading
- Entity#teleport. Under no circumstances will this return;
resort to teleportAsync instead.
- Additional issues may exist.
## Anticipated API Additions
Appropriate asynchronous events. This would facilitate the completion of an event
outcome at a later time, within a distinct thread context. This is requisite
for implementing aspects such as spawn position selection, as asynchronous
chunk loads are necessary when accessing chunk data beyond the region.
World loading/unloading
Further additions will follow.
## Anticipated API Modifications
Exceptionally aggressive thread checks ubiquitously. This is an absolute
necessity to preclude plugin developers from releasing code that might sporadically
disrupt disparate server components in entirely undiagnosable fashions.
Additional changes will be introduced.

### Maven Specifications
* Maven Repo (for foliocopia-api):
```xml
<repository>
    <id>rootbeer</id>
    <url>https://rootbeer.co/maven-public/</url>
</repository>
```
* Artifact Data:
```xml
<dependency>
    <groupId>dev.foliocopia</groupId>
    <artifactId>foliocopia-api</artifactId>
    <version>1.19.4-R0.1-SNAPSHOT</version>
    <scope>provided</scope>
</dependency>
 ```


## Licensing
The PATCHES-LICENSE file elucidates the licensing for API & server patches,
situated in ./patches and its ancillary directories, barring exceptions noted otherwise.

This fork originates from PaperMC's fork exemplar found [here](https://github.com/PaperMC/paperweight-examples).
Consequently, it encompasses alterations to the project. Kindly refer to the repository for licensing information
pertaining to the modified files.
