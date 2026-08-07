---
title: Finding someone without a number plate
date: 2026-08-07
description: What happens to a search problem when there is no unique key to search on, and why the hard part turned out to be memory and graphics cards rather than neural networks.
tags: [systems, python, computer-vision]
---

Ask a computer to find a specific car in a city and it will do it in about a millisecond. That is, if you can read the number plate.

A number plate is a primary key. It is a short string that belongs to exactly one vehicle, so finding that vehicle is just a database lookup. Index the column, type the string, get your answer. Databases have had fifty years of practice at this and they are very good at it.

Now take the plate away. The angle is wrong, or the plate is dirty, or it is raining, or it is night. Or you are not looking for a car at all. You are looking for a person, and people do not have number plates.

That is the interesting version of the problem, and the answer to it shapes everything else you build.

Here is the short version of what follows. When you cannot look something up, you have to describe everything you see, all the time, and search the descriptions instead. That sounds like a machine learning problem. In practice the machine learning was the easy part. The hard part was moving images between processes fast enough to keep up, and the two changes that mattered most were about who owns a piece of memory and about how a graphics card decides which program gets to use it.

## Similar instead of equal

If you cannot match exactly, you match approximately.

A neural network looks at a cropped image and produces a list of a few hundred numbers, called a vector. The network is trained so that two photographs of the same jacket produce two very similar lists, and photographs of different jackets produce lists that are far apart.

Finding something again then becomes a geometry question rather than a text question. Which of the stored vectors sit closest to this one?

You can do the same trick with language. Take a sentence like "dark jacket, backpack, light trousers", turn it into a vector in a comparable space, and now a plain English query can be matched against what the cameras actually recorded.

Two things follow from this, and the second one drives everything.

The first is that answers are ranked, not exact. You get the twenty closest matches, never "this is definitely the one". That is a real change in what the software can honestly promise, and it is worth being upfront about.

The second is that you have to describe everything in advance. You cannot search footage you never described, because searching means comparing against vectors you already computed. So every object in every frame has to be detected, cropped, described and stored, continuously, whether or not anyone ever asks a question about it.

That single requirement quietly converts a search problem into a throughput problem.

## The shape of the system

The pipeline is a funnel. Frames arrive on a queue. A detector finds objects and cuts out crops. Each crop then goes to a set of specialists: one model reads number plates, another produces a vehicle appearance vector, another describes what a person is wearing, another produces a person appearance vector. The results are gathered back into a single record per object and written to a database built for vector search.

The important detail is that the funnel widens before it narrows. One frame becomes many crops, and each crop is needed by several specialists at the same time. Data does not flow down a pipe. It multiplies on the way down.

That matters because of how Python works. Python has a global interpreter lock, which means threads do not give you parallel CPU work. Real parallelism means separate processes. Separate processes do not share memory. And this system's entire job is to move images around, constantly, to several places at once.

## The bottleneck was copying pixels

The obvious way to send an image to another process is to put it on a queue. This looks harmless and is not.

Anything you put on a Python multiprocessing queue gets pickled. It is serialized into a stream of bytes, written into a pipe, read out by the receiving process, and rebuilt into an object on the other side. For a small dictionary that is free. For an image it is not. A fairly modest 640 by 480 colour crop is about 900 kilobytes of raw pixel data, and you pay that cost once for every consumer that needs it.

So the fan-out that makes the system useful is also what makes it expensive. Two specialists means two full copies. Four means four.

The fix is to stop sending the pixels at all.

The image goes into a named block of shared memory, which is a region the operating system will map into several processes at once. The queue then carries only a small note saying where to find it: a name, a shape, and a data type. About a hundred bytes, no matter how large the picture is.

<figure>
<svg viewBox="0 0 720 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;color:inherit" role="img" aria-label="Diagram comparing pickling an image through a queue against passing a shared memory descriptor">
  <defs>
    <marker id="ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>
  <g font-family="system-ui, sans-serif" fill="currentColor">

    <text x="0" y="16" font-size="14" font-weight="600">Before: the image itself travels</text>

    <rect x="2" y="46" width="104" height="42" rx="4" fill="none" stroke="currentColor" stroke-width="1.5"/>
    <text x="54" y="72" font-size="13" text-anchor="middle">Reader</text>

    <rect x="286" y="46" width="92" height="42" rx="4" fill="none" stroke="currentColor" stroke-width="1.5"/>
    <text x="332" y="72" font-size="13" text-anchor="middle">Queue</text>

    <rect x="546" y="16" width="172" height="40" rx="4" fill="none" stroke="currentColor" stroke-width="1.5"/>
    <text x="632" y="41" font-size="13" text-anchor="middle">Plate reader</text>

    <rect x="546" y="78" width="172" height="40" rx="4" fill="none" stroke="currentColor" stroke-width="1.5"/>
    <text x="632" y="103" font-size="13" text-anchor="middle">Appearance model</text>

    <line x1="108" y1="67" x2="282" y2="67" stroke="currentColor" stroke-width="1.5" marker-end="url(#ar)"/>
    <text x="195" y="59" font-size="12" text-anchor="middle" fill="#d9534f">900 KB</text>

    <line x1="380" y1="60" x2="542" y2="38" stroke="currentColor" stroke-width="1.5" marker-end="url(#ar)"/>
    <text x="470" y="38" font-size="12" text-anchor="middle" fill="#d9534f">900 KB</text>

    <line x1="380" y1="76" x2="542" y2="97" stroke="currentColor" stroke-width="1.5" marker-end="url(#ar)"/>
    <text x="470" y="107" font-size="12" text-anchor="middle" fill="#d9534f">900 KB</text>

    <text x="2" y="146" font-size="12.5" opacity="0.75">Serialized once, rebuilt twice. The cost grows with every extra consumer.</text>

    <line x1="0" y1="172" x2="720" y2="172" stroke="currentColor" stroke-width="1" opacity="0.25"/>

    <text x="0" y="204" font-size="14" font-weight="600">After: only a note about the image travels</text>

    <rect x="2" y="234" width="104" height="42" rx="4" fill="none" stroke="currentColor" stroke-width="1.5"/>
    <text x="54" y="260" font-size="13" text-anchor="middle">Reader</text>

    <rect x="286" y="234" width="92" height="42" rx="4" fill="none" stroke="currentColor" stroke-width="1.5"/>
    <text x="332" y="260" font-size="13" text-anchor="middle">Queue</text>

    <rect x="546" y="212" width="172" height="38" rx="4" fill="none" stroke="currentColor" stroke-width="1.5"/>
    <text x="632" y="236" font-size="13" text-anchor="middle">Plate reader</text>

    <rect x="546" y="264" width="172" height="38" rx="4" fill="none" stroke="currentColor" stroke-width="1.5"/>
    <text x="632" y="288" font-size="13" text-anchor="middle">Appearance model</text>

    <line x1="108" y1="255" x2="282" y2="255" stroke="currentColor" stroke-width="1.5" marker-end="url(#ar)"/>
    <text x="195" y="247" font-size="12" text-anchor="middle" fill="#2e9e5b">100 bytes</text>

    <line x1="380" y1="249" x2="542" y2="233" stroke="currentColor" stroke-width="1.5" marker-end="url(#ar)"/>
    <text x="466" y="230" font-size="12" text-anchor="middle" fill="#2e9e5b">100 bytes</text>

    <line x1="380" y1="263" x2="542" y2="281" stroke="currentColor" stroke-width="1.5" marker-end="url(#ar)"/>
    <text x="466" y="292" font-size="12" text-anchor="middle" fill="#2e9e5b">100 bytes</text>

    <rect x="2" y="336" width="716" height="38" rx="4" fill="none" stroke="currentColor" stroke-width="1.5" stroke-dasharray="5 3"/>
    <text x="360" y="360" font-size="13" text-anchor="middle">shared memory block: the pixels, written once, read in place</text>

    <line x1="54" y1="278" x2="54" y2="334" stroke="currentColor" stroke-width="1.2" opacity="0.6" marker-end="url(#ar)"/>
    <line x1="632" y1="304" x2="632" y2="334" stroke="currentColor" stroke-width="1.2" opacity="0.6" stroke-dasharray="4 3"/>
    <text x="646" y="326" font-size="11.5" opacity="0.7">map, do not copy</text>

    <text x="0" y="394" font-size="12.5" opacity="0.75">The picture stops moving. Only its address does.</text>
  </g>
</svg>
</figure>

In code, writing the image looks like this:

```python
def write_image_to_shm(img: np.ndarray) -> dict:
    name = _SHM_PREFIX + uuid.uuid4().hex
    shm = SharedMemory(create=True, name=name, size=img.nbytes)
    try:
        np.ndarray(img.shape, dtype=img.dtype, buffer=shm.buf)[:] = img
    except Exception:
        shm.close()
        shm.unlink()
        raise
    shm.close()  # detach the writer; the block itself stays alive
    return {'shm_name': name, 'shm_shape': img.shape, 'shm_dtype': str(img.dtype)}
```

That is the whole idea. The rest of this section is about why it is harder than it looks.

## Somebody has to own the memory

Shared memory in Python is a two function API. The API is not the difficult part. The difficult part is that you have just introduced manual memory management into a language that was supposed to handle that for you, and the garbage collector cannot see any of it.

Every block needs exactly one owner, and ownership has to move. The rule that worked was simple to say and easy to get wrong: the producer creates the block, writes into it, and detaches immediately. The consumer attaches, copies the pixels out, and deletes the block. The queue is not transferring data. It is transferring responsibility.

```python
if 'shm_name' in data:
    shm = SharedMemory(name=data['shm_name'])
    try:
        arr = np.ndarray(data['shm_shape'], dtype=data['shm_dtype'], buffer=shm.buf)
        return arr.copy()
    finally:
        shm.close()
        shm.unlink()
```

The `finally` block is doing real work there. If decoding throws an exception, the memory still has to be released. Otherwise every failure costs you memory permanently, and failures are not rare when you are processing video around the clock.

## The bug that only appears under load

Here is the one that actually caused trouble.

A single crop often needs to go to two different pools of workers. The tempting move is to write the image into one shared block and put the same note on both queues. Same pixels, half the memory, obviously correct.

It is not correct, and it fails in the least convenient way possible: intermittently, only when busy.

The two workers share a task identifier but run completely independently. The first one finishes, copies its pixels, deletes the block and reports that its share of the task is done. Meanwhile the second worker is still in the middle of attaching. It reaches for a block that has already been destroyed.

<figure>
<svg viewBox="0 0 720 330" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;color:inherit" role="img" aria-label="Timeline showing one shared block being freed by the first worker while the second worker is still attaching, and the fix using one block per worker">
  <g font-family="system-ui, sans-serif" fill="currentColor">

    <text x="0" y="16" font-size="14" font-weight="600">One block, two workers</text>

    <line x1="96" y1="118" x2="700" y2="118" stroke="currentColor" stroke-width="1" opacity="0.3"/>
    <text x="660" y="136" font-size="11.5" opacity="0.6">time</text>

    <text x="0" y="56" font-size="12.5">Worker A</text>
    <rect x="120" y="40" width="90" height="24" rx="3" fill="none" stroke="currentColor" stroke-width="1.4"/>
    <text x="165" y="57" font-size="11.5" text-anchor="middle">attach</text>
    <rect x="216" y="40" width="90" height="24" rx="3" fill="none" stroke="currentColor" stroke-width="1.4"/>
    <text x="261" y="57" font-size="11.5" text-anchor="middle">copy</text>
    <rect x="312" y="40" width="112" height="24" rx="3" fill="none" stroke="#d9534f" stroke-width="1.6"/>
    <text x="368" y="57" font-size="11.5" text-anchor="middle" fill="#d9534f">delete block</text>

    <text x="0" y="100" font-size="12.5">Worker B</text>
    <rect x="392" y="84" width="90" height="24" rx="3" fill="none" stroke="currentColor" stroke-width="1.4" stroke-dasharray="4 3"/>
    <text x="437" y="101" font-size="11.5" text-anchor="middle">attach</text>
    <text x="496" y="101" font-size="12.5" fill="#d9534f">the block is already gone</text>

    <line x1="368" y1="66" x2="410" y2="82" stroke="#d9534f" stroke-width="1.3" stroke-dasharray="3 3"/>

    <text x="0" y="168" font-size="12.5" opacity="0.75">Worker A finished first and cleaned up. It had no way to know B had not started yet.</text>

    <line x1="0" y1="192" x2="720" y2="192" stroke="currentColor" stroke-width="1" opacity="0.25"/>

    <text x="0" y="224" font-size="14" font-weight="600">The fix: one block each, allocated before anyone is told about the task</text>

    <text x="0" y="264" font-size="12.5">Worker A</text>
    <rect x="120" y="248" width="304" height="24" rx="3" fill="none" stroke="#2e9e5b" stroke-width="1.5"/>
    <text x="272" y="265" font-size="11.5" text-anchor="middle">attach, copy, delete its own block</text>

    <text x="0" y="306" font-size="12.5">Worker B</text>
    <rect x="392" y="290" width="304" height="24" rx="3" fill="none" stroke="#2e9e5b" stroke-width="1.5"/>
    <text x="544" y="307" font-size="11.5" text-anchor="middle">attach, copy, delete its own block</text>
  </g>
</svg>
</figure>

The fix is unglamorous. Allocate one block per consumer, and allocate all of them before the task is announced anywhere. If the second allocation fails, release the first and drop the task cleanly. No worker is ever holding a reference to memory that another worker is entitled to free.

Bugs like this do not appear in testing, because in testing the workers are not competing for anything. They appear at three in the morning at peak load, and they look like random corruption rather than a logic error.

## Crashes leave litter

On Linux, shared memory lives in a filesystem that exists in RAM. The blocks are owned by the kernel, not by your program, which means they outlive the process that created them. If a worker is killed halfway through a task, Python exiting does not clean anything up. The block just sits there holding memory.

Do that a few thousand times and the machine fills up with orphaned images and starts behaving strangely in ways that look nothing like the original crash.

The defence is to give every block a name with a known prefix, and sweep on startup:

```python
def cleanup_leaked_shm() -> int:
    count = 0
    for path in glob.glob(f"/dev/shm/{_SHM_PREFIX}*"):
        try:
            os.unlink(path)
            count += 1
        except Exception:
            pass
    return count
```

That is trivial code, and it only works because of the naming convention. The prefix is what lets the sweep be confident it is deleting things this program created and nothing else.

## Two processes, one graphics card

The other change worth writing about had nothing to do with memory.

Several worker processes shared a single GPU. The reasonable assumption is that they use it at the same time. That is not what happens.

By default, when multiple processes submit work to one CUDA device, the driver gives them the card in turns. Process one gets exclusive use for a slice of time, then yields, then process two gets its slice, and so on. Only one process's work is running at any given instant.

For a lot of workloads that is fine, because one process running a large batch will already fill the card. It was very much not fine here.

The work in this pipeline is many small pieces. A single cropped image through a small detection model uses a tiny fraction of a modern GPU, which has thousands of cores. So during its turn, a process would occupy a sliver of the chip and leave the rest idle, while the other processes waited their turn to also not fill it. The card was busy in the sense that something was always scheduled on it, and almost empty in the sense that almost none of it was doing arithmetic.

Picture an eight lane motorway where the traffic rules allow one car at a time.

<figure>
<svg viewBox="0 0 720 340" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;color:inherit" role="img" aria-label="Timeline comparing GPU time slicing, where only one process runs at a time, with concurrent execution where kernels from different processes overlap">
  <g font-family="system-ui, sans-serif" fill="currentColor">

    <text x="0" y="16" font-size="14" font-weight="600">Default: the processes take turns</text>

    <text x="0" y="52" font-size="12">Process 1</text>
    <text x="0" y="82" font-size="12">Process 2</text>
    <text x="0" y="112" font-size="12">Process 3</text>

    <line x1="78" y1="126" x2="700" y2="126" stroke="currentColor" stroke-width="1" opacity="0.3"/>
    <text x="662" y="144" font-size="11.5" opacity="0.6">time</text>

    <rect x="84" y="38" width="66" height="20" rx="3" fill="currentColor" opacity="0.75"/>
    <rect x="318" y="38" width="66" height="20" rx="3" fill="currentColor" opacity="0.75"/>
    <rect x="552" y="38" width="66" height="20" rx="3" fill="currentColor" opacity="0.75"/>

    <rect x="162" y="68" width="66" height="20" rx="3" fill="currentColor" opacity="0.75"/>
    <rect x="396" y="68" width="66" height="20" rx="3" fill="currentColor" opacity="0.75"/>
    <rect x="630" y="68" width="66" height="20" rx="3" fill="currentColor" opacity="0.75"/>

    <rect x="240" y="98" width="66" height="20" rx="3" fill="currentColor" opacity="0.75"/>
    <rect x="474" y="98" width="66" height="20" rx="3" fill="currentColor" opacity="0.75"/>

    <text x="0" y="168" font-size="12.5" opacity="0.75">Something is always scheduled. Most of the chip is idle anyway, because each small job uses a sliver of it.</text>

    <line x1="0" y1="192" x2="720" y2="192" stroke="currentColor" stroke-width="1" opacity="0.25"/>

    <text x="0" y="224" font-size="14" font-weight="600">With a scheduler in front: the work overlaps</text>

    <text x="0" y="260" font-size="12">Process 1</text>
    <text x="0" y="290" font-size="12">Process 2</text>
    <text x="0" y="320" font-size="12">Process 3</text>

    <rect x="84" y="246" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="162" y="246" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="240" y="246" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="318" y="246" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>

    <rect x="84" y="276" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="162" y="276" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="240" y="276" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="318" y="276" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>

    <rect x="84" y="306" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="162" y="306" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="240" y="306" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>
    <rect x="318" y="306" width="66" height="20" rx="3" fill="#2e9e5b" opacity="0.85"/>

    <line x1="400" y1="240" x2="400" y2="330" stroke="currentColor" stroke-width="1.2" stroke-dasharray="4 3" opacity="0.55"/>
    <text x="412" y="290" font-size="12.5" opacity="0.8">same work, finished here</text>
  </g>
</svg>
</figure>

The fix is a piece of NVIDIA plumbing called the Multi-Process Service. It runs as a small background daemon, and instead of each process talking to the driver on its own, they all submit work through it. Because everything arrives through a single channel, the card can run kernels from different processes genuinely side by side, on different parts of the chip, at the same time.

What I like about this one is the ratio of effort to result. There was no change to any of the worker code. Start a daemon, set an environment variable so the processes know to use it, and the same models on the same hardware produced roughly double the throughput. Utilisation went from embarrassing to nearly full.

It is worth knowing when this does not help. If you have one process feeding the GPU large batches, it is already filling the card and this buys you nothing. The technique is specifically for the case where a lot of small pieces of work are arriving from separate processes, which is exactly the shape of a pipeline built around Python's process model.

There is also a trap worth mentioning, because it caught me. On a different machine the same environment variable had to be set in the opposite direction. A service was already running there for an unrelated workload, and our processes were being funnelled into it by default, which we did not want. Same knob, same value, opposite intent. Worth checking which situation you are actually in before you copy a configuration between machines.

## What was actually hard

Almost none of this was machine learning. The models were off the shelf and most of them worked more or less on the first try.

The time went somewhere else. It went into deciding who owns a buffer, into making sure two workers could never free the same one, into cleaning up memory that survives the program that created it, and into discovering that a graphics card with several processes on it had been quietly running them one at a time.

None of that is visible in a demo. All of it is the difference between a system that works on a laptop and one that keeps up with a city.
