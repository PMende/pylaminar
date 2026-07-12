# pylaminar
A modern, Pythonic, idiomatic orchestration engine that prioritizes a smooth developer experience over all else. This project is pre-alpha, and not meant for use as of last update (`2026-07-12`).

## Design philosophy
A [laminar flow](https://en.wikipedia.org/wiki/Laminar_flow) is a concept from physics (specifically fluid dynamics) that distinguishes between turbulent and smooth (or laminar) flows of a fluid.  This is the origin of `pylaminar`'s name: workflow orchestration frameworks available to data practitioners (data scientists, machine learning engineers, data/analytics engineers, among others) today tend to prioritize *capabilities* rather than developer ergonomics. The result is that the development process of data science/engineering and ML workflows often eventually become "turbulent". Even in frameworks where developer ergonomics are a prominent target (e.g. [Prefect](https://www.prefect.io/)).

`pylaminar` takes the position that developer ergonomics/experience comes first. Always. If a new capability is required, its implementation **must** admit an API that is, at minimum:
- Trivially understable at the API layer for the typical practitioner it targets
- Runnable locally

No new capability is considered complete until the above is true. As a result, complexity *must* be consumed internally by `pylaminar` itself: never by the end-user.

## Installing
Install this project (**in the future**) via [`pixi`](https://pixi.prefix.dev/latest/):

```bash
pixi install pylaminar
```

## Contributing
Contributors are encouraged to develop as they see fit so long as the design and implementation of new features conforms to the norms of the project. If you make use of AI-assisted development, you're ecnouraged to make use of our [`CLAUDE.md`](./CLAUDE.md)

## Quickstart
TBD

## Usage
TBD

## Architecture
TBD