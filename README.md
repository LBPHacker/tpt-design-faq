# TPT design decisions FAQ

## <a name="good-cpu"></a> I have a very good CPU, why are frame rates so low in general?

It's because these days, when CPU manufacturers have shifted their focus from single-core performance to efficiency and core count, a single one of your "very good" CPU's cores is probably about as powerful as a high-end single-core CPU from 2008, and since TPT does the majority of calculations on a single thread, and thus a single core, you get frame rates appropriate for such a CPU. See [Is TPT multithreaded?](#multithread-pls)

## <a name="multithread-pls"></a> Is TPT multithreaded?

Some tasks do run on their own, separate threads, but mostly, no, it isn't.

The following tasks run on separate threads:

 - Newtonian gravity simulation; you can see the effects of this running on a separate thread when you set up a path that includes Newtonian gravity elements (`WHOL`, `BHOL`, etc.) for energy particles (`PHOT`, etc.) to follow, and the stream of particles wavers slightly at times;
 - rendering of paste previews; custom elements don't show up in these previews because it's difficult to let the other thread know about custom elements;
 - a few others that aren't too closely related to the simulation.
 
Everything else runs on the main thread. Yeah. See [Why isn't TPT fully multithreaded?](#why-not-fully-multithreaded)

## <a name="good-gpu"></a> I have a very good GPU, why are frame rates so low when in Fancy display?

Because TPT doesn't use the GPU to render all those fancy effects. It renders them on the CPU on the same thread, and thus the same core, that it does almost everything else on. See [Does TPT use the GPU?](#ghwaccel-pls)

## <a name="ghwaccel-pls"></a> Does TPT use the GPU?

Not really, beyond some very basic functions. All rendering is done on the CPU.

There have been attempts in the past to offload rendering from the CPU to the GPU, and remains of these attempts still linger in the codebase today. Sadly, GPUs were not that good at rendering individual dots from a list of dots to render, and it had long been established that CPUs where not that good at rendering all the effects TPT had. Hybrid approaches have not been considered due to the technical difficulties involved. Worse, these attempts have been made back when not everyone had a GPU that could support them, so the GPU rendering code had to be maintained in tandem with the CPU rendering code. Over time, this GPU code withered and got disabled, and is not available in-game today.

These days, now that virtually every consumer computer in existence and in daily use has a GPU powerful and featureful enough to handle TPT's effects, it might be feasible to offload all rendering to the GPU, even if it's some low profile integrated one. This has not been attempted due to lack of time, see [Why is TPT development so slow?](#lazy-devs)

## <a name="why-not-fully-multithreaded"></a> Why isn't TPT fully multithreaded?

If we interpret the goal to "multithread TPT" to mean spreading simulation calculations out on more cores to achieve higher frame rates, we face a choice among the following set of options:

 - upgrade the current simulation to run on multiple threads, limiting ourselves to changes that preserve compatibility;
   - there doesn't seem to be too much we can do here; many people, including very smart ones with a lot of time, have tried and failed to do this over the last decade;
   - this of course doesn't mean that it's not possible, but there's a limit to how much time and effort we can put into looking for solutions, see [Why is TPT development so slow?](#lazy-devs)
 - upgrade the current simulation to run on multiple threads, accepting the consequences of breaking compatibility;
   - this is promising because it'd likely mean little to no difference visible to users in practice most of the time, obviously beside the lack of compatibility with a select few mechanics;
   - however, the lack of compatibility with more important mechanics might prove annoying; the effect of such breakage of compatibility is difficult to assess, you can't easily just ask the community about it and expect comprehensive and informed answers;
   - this hasn't been throughly explored because once you are prepared to break compatibility, you might as well save yourself from the burden of old code and just go with the next option;
 - re-design the simulation to run on multiple threads, providing the old simulation as a fallback for old saves;
   - this option can afford disregarding compatibility completely because two relatively independent simulations would exist in the codebase, only one of which would need to be actively maintained;
   - this is by far the most promising option;
   - sadly TPT is difficult to multithread even if you do have the time and are prepared to break compatibility, see [Why are particle simulations like TPT so difficult to multithread?](#multithreading-hard)

## <a name="multithreading-hard"></a> Why are particle simulations like TPT so difficult to multithread?

TODO. This is what I'll be doing next, I swear xd

## <a name="lazy-devs"></a> Why is TPT development so slow?

The game, the simulation server, and all the other services, such as the snapshot server and TPTMP, are being developed and maintained mainly by two people, [jacob1] and [LBPHacker] (but also [a ton of contributors][contributors], thank you all), in their free time, for free. This combination puts a very hard limit on how much time and effort is spent on the project, and the rate of development reflects this.

[jacob1]: https://powdertoy.co.uk/User.html?Name=jacob1
[LBPHacker]: https://powdertoy.co.uk/User.html?Name=LBPHacker
[contributors]: https://github.com/The-Powder-Toy/The-Powder-Toy/graphs/contributors
