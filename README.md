# TPT design decisions FAQ

## <a name="good-cpu"></a> I have a very good CPU, why are frame rates so low in general?

It's because these days, when CPU manufacturers have shifted their focus from single-core performance to efficiency and core count, a single one of your "very good" CPU's cores is probably about as powerful as a high-end single-core CPU from 2008, and since TPT does the majority of calculations on a single thread, and thus a single core, you get frame rates appropriate for such a CPU. See [Is TPT multithreaded?](#multithread-pls)

## <a name="multithread-pls"></a> Is TPT multithreaded?

Some tasks do run on their own, separate threads, but mostly, no, it isn't.

The following tasks run on separate threads:

 - Newtonian gravity simulation; you can see the effects of this running on a separate thread when you set up a path that includes Newtonian gravity elements (`WHOL`, `BHOL`, etc.) for energy particles (`PHOT`, etc.) to follow, and the stream of particles takes an entire frame to react to changes in the path;
 - rendering, in some cases (if no elements with Lua graphics functions, no Lua rendering event handlers, etc. are registered);
 - rendering of paste previews; Lua elements don't show up properly all the time in these previews because it's difficult to let the other thread know about Lua elements;
 - a few others that aren't too closely related to the simulation.
 
Everything else runs on the main thread. Yeah. See [Why isn't TPT fully multithreaded?](#why-not-fully-multithreaded)

## <a name="good-gpu"></a> I have a very good GPU, why are frame rates so low when in Fancy display?

Because TPT doesn't use the GPU to render all those fancy effects. It renders them on the CPU, and in some cases, it even does it on the same thread, and thus the same core, that it does almost everything else on. Even in the best case, it still uses the CPU to render, albeit a separate thread on it. See [Does TPT use the GPU?](#ghwaccel-pls)

## <a name="ghwaccel-pls"></a> Does TPT use the GPU?

Not really, beyond some very basic functions, and not even those explicitly. A library we rely on, SDL, decides whether to use it. All simulation rendering is done on the CPU.

There have been attempts in the past to offload rendering from the CPU to the GPU, and remains of these attempts lingered in the codebase for years. Sadly, GPUs were not that good at rendering individual dots from a list of dots to render, and it had long been established that CPUs where not that good at rendering all the effects TPT had. Hybrid approaches have not been considered due to the technical difficulties involved. Worse, these attempts have been made back when not everyone had a GPU that could support them, so the GPU rendering code had to be maintained in tandem with the CPU rendering code. Over time, this GPU code withered and got removed, and is not available in-game today.

These days, now that virtually every consumer computer in existence and in daily use has a GPU powerful and featureful enough to handle TPT's effects, it might be feasible to offload all rendering to the GPU, even if it's some low profile integrated one. This has not been attempted due to lack of time, see [Why is TPT development so slow?](#lazy-devs)

## <a name="why-not-fully-multithreaded"></a> Why isn't TPT fully multithreaded?

If we interpret the goal to "multithread TPT" to mean spreading simulation calculations out on more cores to achieve higher frame rates, we face a choice among the following set of options:

 - upgrade the current simulation to run on multiple threads, limiting ourselves to changes that preserve compatibility;
   - there doesn't seem to be too much we can do here; many people, including very smart ones with a lot of time, have tried and failed to do this over the last decade;
   - this of course doesn't mean that it's not possible, but there's a limit to how much time and effort we can put into looking for solutions, see [Why is TPT development so slow?](#lazy-devs)
 - upgrade the current simulation to run on multiple threads, accepting the consequences of breaking compatibility;
   - this is promising because it'd likely mean little to no difference visible to users in practice most of the time, obviously beside the lack of compatibility with a select few mechanics;
   - however, the lack of compatibility with more important mechanics might prove annoying; the effect of such breakage of compatibility is difficult to assess, you can't easily just ask the community about it and expect comprehensive and informed answers;
   - this hasn't been thoroughly explored because once you are prepared to break compatibility, you might as well save yourself from the burden of old code and just go with the next option;
 - re-design the simulation to run on multiple threads, providing the old simulation as a fallback for old saves;
   - this option can afford disregarding compatibility completely because two relatively independent simulations would exist in the codebase, only one of which would need to be actively maintained;
   - this is by far the most promising option;
   - sadly TPT is difficult to multithread even if you do have the time and are prepared to break compatibility, see [Why are particle simulations like TPT so difficult to multithread?](#multithreading-hard) and [Why is TPT so difficult to multithread?](#multithreading-tpt-hard)

## <a name="multithreading-hard"></a> Why are particle simulations like TPT so difficult to multithread?

In this context, "particle simulations like TPT" means simulations with at least the following features relevant to the discussion:

 - discrete particles: particles are individual entities, not some artefact of the visualization of a vector field of some fluid dynamics process;
   - [cool fluid sim][webglfluid] that does not have this feature;
 - integer grid: every particle is at least virtually aligned to a grid with integer resolution, meaning that even if a particle is at a non-integer position, it can be treated as if it is by integer-grid processes such as collision detection and rendering;
   - [cool particle sim][ndparticles] that does not have this feature;
 - particle-particle interactions: particles interact not only with environmental factors (such as forces from air and gravity simulation) but with one another as well;
   - [very old AMD demo][prototptvideo] that does not have this feature;
 - no self-stacking: particles don't normally move on top of one another ("stack"); they may do this by their own choice in exceptional cases, but certainly not by default;
   - the demo in the previous point does not have this feature either;
 - non-local interactions: interactions whose scope extends beyond the local neighbourhood of particles; think INST or PSTN.

TPT has all of these, as do probably many other falling sand-type games (FSGs) out there, citation needed. TPT doesn't have a number of features that we would consider good to have for this type of game, but which would complicate matters even further:

 - determinism: a simulation that goes from an initial state to a final state after a certain number of simulated frames and user interaction always goes to that final state after the same number of simulated frames and the same user interaction, regardless of application version, operating system, machine architecture, etc.
   - differences in the final state between application versions may be allowed, but this must be recognized and advertised as a breaking change; there should be as few of these as possible;
 - extensibility: the sky is the limit;
   - TPT's simulation is extensible enough to serve the needs of most people who just want to add an element with a few interactions, and not much beyond that.

Note that none of these features concerns realism, usability, or performance.

Particle simulations like TPT in general are difficult to multithread because there are many non-local interactions that affect many particles at once, interfering with the rest of the simulation. Further, the effects of local interactions regularly overlap, causing even these to interfere with one another. Interference is the bane of parallelization, and thus multithreading.

TPT completely avoids the problem of interference by not even trying to be parallel: every single interaction between particles and their environment is executed in series, in a single total order from the point of view of all particles and all environmental factors. This is what we call subframe mechanics, and they are what's behind many kinds of bias in the simulation.

## <a name="multithreading-tpt-hard"></a> Why is TPT so difficult to multithread?

Lua is often chosen as a scripting language in games for its simplicity and ease of integration, and especially [LuaJIT][luajit], a very fast implementation of Lua, is very common; indeed, official builds of TPT use LuaJIT.

On top of everything discussed in the previous section, TPT intends Lua to be used for two purposes by way of providing appropriate APIs: implementing simulation manipulation and introspection tools (terrain generators, reorderers, custom HUDs, etc.) and implementing simulation mechanics (custom elements, custom reactions, etc.). The runtime performance of custom tools is not central to the overall experience; they can very well be very slow and still be practical. The runtime performance of custom simulation mechanics, however, is very much central for an FSG.

There are several ways in which simulation mechanics are prevented from achieving high performance in practice by the choice of Lua as TPT's scripting language and by the design of its APIs:

 - Lua is too powerful: it's a general-purpose language with general-purpose primitives
   - associative data structure manipulation, function calls, closures, string manipulation; these are practically useless for an FSG;
 - Lua is not powerful enough: its minimal nature makes it impossible for its primitives to be extended as required by the use case;
   - primitives, such as particle property access and other simulation state access would be ideal for an FSG, but these have to be implemented with Lua's general-purpose primitives instead, invoking substantial overhead;
 - Lua (read: relevant implementations) is not intended for multi-threaded use: this immediately very strongly limits what can be done on other threads;
   - as an example, separate thread rendering of the simulation is only enabled if this wouldn't involve calling Lua functions; that is, it is disabled if e.g. there is a single particle in the entire simulation with a custom Graphics function;
 - the API is too powerful: it puts very few limits on what is and isn't allowed to be done;
   - this implies that anything *may* be done, which in turn *causes* everything to be done;
   - this locks the API into very specific ways of operation in the name of compatibility;
 - the API is not powerful enough: some very obvious ways of interacting with the simulation are incomplete or missing;
   - a common example of this is custom particle data (that which is beyond the storage capability of individual particle properties; think WIFI activation data, GOL state, PRTI/PRTO particle storage);
   - the incomplete state or lack of these abilities justifies the aforementioned lack of limits imposed on usage.

A proposed solution for all of these problems is an entirely custom language for implementing simulation mechanics. This could coexist with support for the existing Lua API, with recommendations against the Lua API and for the new language in place. Naturally, this would be a huge undertaking and has not gained much traction, see [Why is TPT development so slow?](#lazy-devs)

## <a name="lazy-devs"></a> Why is TPT development so slow?

The game, the simulation server, and all the other services, such as the snapshot server and TPTMP, are primarily being developed and maintained mainly by two people, [jacob1] and [LBPHacker] \(and to a lesser extent also [a ton of contributors][contributors], thank you all), in their free time, for free. This combination puts a very hard limit on how much time and effort is spent on the project, and the rate of development reflects this.

[jacob1]: https://powdertoy.co.uk/User.html?Name=jacob1
[LBPHacker]: https://powdertoy.co.uk/User.html?Name=LBPHacker
[contributors]: https://github.com/The-Powder-Toy/The-Powder-Toy/graphs/contributors
[webglfluid]: https://paveldogreat.github.io/WebGL-Fluid-Simulation/
[ndparticles]: https://david.li/fluid/
[prototptvideo]: https://www.youtube.com/watch?v=7PAiCinmP9Y
[luajit]: https://luajit.org/
