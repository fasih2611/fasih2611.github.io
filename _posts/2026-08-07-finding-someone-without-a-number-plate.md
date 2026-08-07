---
title: Finding someone without a number plate
date: 2026-08-07
description: What happens to a search problem when there is no unique key to search on, why such a system needs several models rather than one, and the systems work underneath them.
tags: [systems, python, computer-vision]
---

Ask a computer to find a specific car in a city and it will do it in about a millisecond. That is, if you can read the number plate.

A number plate is a primary key. It is a short string that belongs to exactly one vehicle, so finding that vehicle is a database lookup. Index the column, type the string, get your answer. Databases have had fifty years of practice at this and they are very good at it.

Now take the plate away. The angle is wrong, or the plate is dirty, or it is raining, or it is night. Or you are not looking for a car at all. You are looking for a person, and people do not have number plates.

That is the interesting version of the problem, and the answer to it determines everything else you build.

This post is about two things that surprised me while building such a system. The first is that it needs a lot of models rather than one, and the arrangement of those models matters more than any individual choice among them. The second is that once the models worked, almost all of the remaining difficulty was in moving images between processes and in getting a graphics card to do what I thought it was already doing.

## Similar instead of equal

If you cannot match exactly, you match approximately.

A neural network looks at a cropped image and produces a list of a few hundred numbers, called a vector. The network is trained so that two photographs of the same object produce two similar lists, and photographs of different objects produce lists that are far apart.

Finding something again is then a geometry question rather than a text question. Which of the stored vectors are closest to this one?

Two things follow. The first is that answers are ranked, not exact. You get the twenty closest matches, never "this is definitely the one". That changes what the software can honestly promise and it is worth being clear about.

The second is that you have to describe everything in advance. You cannot search footage you never described, because searching means comparing against vectors you already computed. Every object in every frame has to be detected, cropped, described and stored, continuously, whether or not anyone ever asks a question about it.

That requirement is what turns a search problem into a throughput problem.

## Why there are several models and not one

The obvious design is one model that looks at a frame and returns everything you want to know about it. That design works in a demo and gets very expensive very quickly.

The reason is that the useful models are expensive per object, and most of a frame contains nothing you care about. So the work is arranged as a cascade. Cheap and broad at the top, expensive and narrow at the bottom, with every stage reducing how much work the stage below it has to do.

Almost every decision below follows from that one idea.

### Finding things at all

The top of the cascade is a single detector that runs on every frame and finds objects worth looking at more closely.

I use one detector for both vehicles and people rather than a specialised one for each. Running a detector over a frame costs about the same whether it is looking for one class or several, so two detectors would roughly double the cost of the most frequently executed stage in the system to buy very little.

This stage sees every pixel of every frame, so it has to be fast, and in exchange it is allowed to be imprecise. Its only job is to narrow the field.

### Finding the small thing inside the big thing

A number plate takes up a tiny fraction of a full frame. Running a plate detector across the whole image is both expensive and bad at its job, because at full-frame scale the plate is only a handful of pixels across and detection quality falls apart.

So plate detection does not run on frames. It runs on the vehicle crops produced by the previous stage. The crop is a few hundred pixels across rather than a few thousand, which means the plate inside it is now comparatively large, and the detector only ever runs on regions already known to contain a vehicle.

The same model becomes both more accurate and cheaper, purely because of what it is shown. That is the cascade earning its keep, and it is worth internalising as a general technique: when a model underperforms on small objects, the fix is often not a better model but a tighter crop.

### Reading text, and a stage I deleted

Text recognition is conventionally two models. The first finds the regions of an image that contain text. The second reads the characters inside each region.

I was running both, and the first was doing nothing useful. Its input was already a tight crop of a plate, produced by the plate detector one stage earlier. "Where is the text in this image of a number plate" is a question that had already been answered upstream.

Deleting that stage took text reading from roughly 250 milliseconds to roughly 25. The recognition model did not change. I had been paying, on every single plate, for an answer I already had.

I think this is a common shape of waste in machine learning pipelines and it is worth describing carefully. Off-the-shelf models arrive packaged as complete solutions, and the packaging assumes it is being handed a raw photograph. When your input is not a raw photograph, part of the package is redundant. Nothing about it looks wrong. There is no error, no warning, and the output is correct. It just quietly does work you do not need, forever.

One more decision at this stage: recognition runs as separate models per script, rather than one model covering all of them. Plates in this setting can carry more than one writing system, and a single model spanning several alphabets is both larger and less accurate at each of them than a specialist. Since the cascade has already narrowed the input to a single plate, running the right specialist is cheap.

### Describing appearance as a vector

This is the stage that answers the original question, and there are two of them: one for vehicles and one for people.

Both are trained with metric learning, which means the training objective is not "what class is this" but "make these two images of the same object produce nearby vectors, and push different objects apart". The output is not a label. It is a coordinate.

I keep vehicles and people on separate models because the invariances they need are different, almost opposite. A useful vehicle representation has to be robust to viewing angle while remaining sensitive to colour and shape. A useful person representation has to be robust to pose and to a person's body turning, while remaining sensitive to clothing. Training one model to do both means asking it to be invariant and sensitive to overlapping things, and you get a model that is mediocre at each.

### Describing appearance in words

There is one more description stage, and it is the most expensive thing in the system by a wide margin: a vision and language model that looks at a crop of a person and writes down what they are wearing.

The reasonable question is why this exists at all when the previous stage already produces a vector.

They answer different questions. An appearance vector answers "find me more of this", and to ask it you need an example image already in hand. Words answer "find me something matching this description", which you can ask with no example at all. If someone can only tell you what a person was wearing, the vector model is useless and the language model is the entire product.

The cost is real. This model is orders of magnitude more expensive per crop than any of the others, and it is the throughput ceiling of the whole pipeline. It only remains viable because it sits at the bottom of the cascade, where it sees a small fraction of what the detector saw.

### The search side

At query time the same principle applies in reverse.

A text query is turned into a vector by a text embedding model and compared against everything stored. That comparison is cheap and approximate, which is what makes it possible to run it against a very large index.

Cheap and approximate also means the ordering it produces is mediocre. So the top results, and only the top results, go through a second model that looks at the query and one candidate together and scores how well they match. Judging a pair jointly is far more accurate than comparing two independently computed vectors, and far too slow to run against everything.

Retrieve cheaply, then rerank expensively, on a small set. It is the same cascade as the ingest side, pointed the other way.

## The bottleneck was copying pixels

With the models settled, the remaining problem was feeding them, and this turned out to be most of the work.

The pipeline is a funnel that widens before it narrows. One frame becomes many crops, and each crop is needed by several of the stages above at the same time. Data does not flow down a pipe. It multiplies on the way down.

That matters because of how Python works. Python has a global interpreter lock, so threads do not give you parallel CPU work. Real parallelism means separate processes, and separate processes do not share memory.

The obvious way to send an image to another process is to put it on a queue. This looks harmless and is not.

Anything placed on a Python multiprocessing queue gets pickled. It is serialised into bytes, written into a pipe, read out by the receiving process, and rebuilt on the other side. For a small dictionary that is free. For an image it is not. A 640 by 480 colour crop is about 900 kilobytes of raw pixels, and you pay that cost once per consumer. The fan-out that makes the system useful is also what makes it expensive.

The fix is to stop sending pixels. The image goes into a named block of shared memory, which the operating system maps into several processes at once, and the queue carries only a note saying where to find it: a name, a shape, and a data type. About a hundred bytes, regardless of image size.

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

    <text x="2" y="146" font-size="12.5" opacity="0.75">Serialised once, rebuilt twice. The cost grows with every extra consumer.</text>

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

Writing the image looks like this:

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

That is the whole idea. The rest of this section is why it is harder than it looks.

## Somebody has to own the memory

Shared memory in Python is a two function API. The API is not the difficult part. The difficult part is that you have introduced manual memory management into a language that was supposed to handle it for you, and the garbage collector cannot see any of it.

Every block needs exactly one owner and ownership has to move. The rule I settled on: the producer creates the block, writes into it, and detaches immediately. The consumer attaches, copies the pixels out, and deletes the block. The queue is not transferring data, it is transferring responsibility.

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

The `finally` is doing real work. If decoding throws, the memory still has to be released, otherwise every failure costs memory permanently. Failures are not rare when you are processing video continuously.

## The bug that only appears under load

A single crop often needs to go to two different pools of workers. The tempting move is to write the image into one shared block and put the same note on both queues. Same pixels, half the memory, apparently correct.

It is not correct. The two workers share a task identifier but run independently. The first one finishes, copies its pixels, deletes the block, and reports its share of the task complete. The second worker is still in the middle of attaching, and reaches for a block that has already been destroyed.

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

The fix is to allocate one block per consumer, and to allocate all of them before the task is announced anywhere. If the second allocation fails, release the first and drop the task cleanly. No worker ever holds a reference to memory another worker is entitled to free.

This class of bug does not show up in testing, because in testing the workers are not competing for anything. It shows up under load, and it presents as random corruption rather than as a logic error, which is what makes it expensive to find.

## Crashes leave litter

On Linux, shared memory lives in a filesystem that exists in RAM, and the blocks are owned by the kernel rather than by your program. They outlive the process that created them. If a worker is killed halfway through a task, Python exiting cleans up nothing. The block sits there holding memory.

Do that a few thousand times and the machine fills with orphaned images and starts behaving strangely in ways that look nothing like the original crash.

The defence is to give every block a name with a known prefix and sweep on startup:

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

Trivial code, and it only works because of the naming convention. The prefix is what lets the sweep be confident it is deleting things this program created and nothing else.

## Two processes, one graphics card

Several worker processes shared a single GPU. The reasonable assumption is that they use it simultaneously. That is not what happens.

By default, when multiple processes submit work to one CUDA device, the driver gives them the card in turns. One process gets exclusive use for a slice of time, then yields, then the next process gets its slice. Only one process's work runs at any instant.

For many workloads this is fine, because a single process running a large batch already fills the card. It was not fine here. The work in this pipeline is many small pieces, and one cropped image through one of the smaller models uses a tiny fraction of a GPU with thousands of cores. During its turn a process would occupy a sliver of the chip and leave the rest idle, while the other processes waited their turn to also not fill it. The card was busy in the sense that something was always scheduled on it, and nearly empty in the sense that almost none of it was doing arithmetic.

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

The fix is a piece of NVIDIA plumbing called the Multi-Process Service. It runs as a background daemon, and instead of each process talking to the driver independently, they all submit work through it. Because everything arrives through one channel, the card can run work from different processes side by side, on different parts of the chip, at the same time.

No worker code changed. Start a daemon, set an environment variable so the processes use it, and the same models on the same hardware produced roughly double the throughput.

It is worth knowing when this does not help. If one process is feeding the GPU large batches, it is already filling the card and this buys nothing. The technique is specifically for many small pieces of work arriving from separate processes, which is the shape you get when Python's process model forces you into multiprocessing.

There is also a trap. On another machine the same environment variable had to be set in the opposite direction, because a service was already running there for an unrelated workload and our processes were being funnelled into it by default. Same knob, same value, opposite intent. Check which situation you are in before copying configuration between machines.

## Wrapping up

If I had to compress this into one idea, it would be the cascade. Losing the primary key means you cannot look anything up on demand, so you have to describe everything continuously, and describing everything is only affordable if each stage is cheap enough to run at the volume it sees. Everything above follows from that constraint, including the model choices, the shared memory, and the GPU scheduling.

The models were the part I expected to be hard, and they mostly were not. They were off the shelf and most of them worked more or less immediately. What took the time was arranging them so the expensive ones ran rarely, and then getting the data to them without copying it four times on the way.
