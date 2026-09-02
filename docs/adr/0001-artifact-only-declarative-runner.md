# Keep PEST artifact-only and declarative

PEST will be a standalone CLI that tests immutable plug-in artifacts through a small declarative manifest; it will not build plug-ins or link their frameworks. This keeps results tied to exact shipped binaries and avoids turning a reusable host tester into another build system, at the cost of requiring each plug-in project to produce and identify its artifacts before invoking PEST.
