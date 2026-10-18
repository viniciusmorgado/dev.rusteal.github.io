# Roadmap

Unreal is a huge engine, and it is constantly changing, as the [UE6 radar](https://github.com/viniciusmorgado/rusteal/blob/ue6/docs/ue6-radar.md) shows. Because of this, we prioritize the bindings and features Rusteal adds by what real projects need, and by what parity with C++ basic functionality requires.

That is why the first features Rusteal brought bindings for were those the engine's own templates use, the ones Unreal offers when a new game is created. Epic already delivers the base boilerplate for a number of genres as part of the engine, and most projects start from it; if Rusteal wants to give its bindings the same experience, the templates have to be ported first. They are the [Rusteal templates](https://viniciusmorgado.github.io/game.rusteal.github.io/getting_started/templates.html), every one of the engine's game templates with its gameplay in Rust.

### Going forward

Right now, the main guide to Rusteal's development is Noir Blue, a first person shooter that the developers of Rusteal itself are making with it: what the game needs and Rusteal lacks is what comes next.

Next comes the road to 1.0, in the repository's [`docs/roadmap-1.0.md`](https://github.com/viniciusmorgado/rusteal/blob/develop/docs/roadmap-1.0.md): what Rusteal lacks against Unreal Engine 5.8 and the milestones that close it. Its first milestone has started with [the launcher](architecture/launcher.md), which brings Rusteal into the editor's New Project dialog and is the way to Fab.

Consoles are planned in two phases. The first, built on desktop and in public, is done (Rusteal 0.9): the module contract, the static backend (`rusteal package --static`), target platforms in the CLI and the runtime's services taken from the engine. The second is the bring-up with a console's SDK, in a private platform pack, once a project has access. See [Consoles](platforms/consoles.md).

What Rusteal does not cover yet, and what each gap takes, is tracked in the repository's `docs/`: the [third person gaps](https://github.com/viniciusmorgado/rusteal/blob/main/docs/third-person-gaps.md), the [plugin roadmap](https://github.com/viniciusmorgado/rusteal/blob/main/docs/plugin-roadmap.md), and, for platforms, mobile and console support, both being worked on (see [Unreal Engine and Rust](intro/unreal_rust.md) for the console path).
