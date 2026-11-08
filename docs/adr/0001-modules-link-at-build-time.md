# Modules link at build time; only render backends and Applications load at runtime

Every Ember Module is its own dynamic library, but Modules are linked like ordinary shared libraries and call each other through plain exported C functions. Only render backends and Applications (a game or the editor) are loaded at runtime, through one entry function that checks the Ember version and returns a typed function table. Modules are chosen when an Application is built and never hot-reload, so runtime loading would add a function table to every call between Modules without buying anything Ember needs. Render backends must load at runtime so one build can pick among them at startup and fall back when a graphics driver is missing. Applications must load at runtime so they can hot-reload.

## Considered Options

- **Every Module loaded at runtime through an entry function that returns an API table.** Rejected: its benefits (swapping Modules by config, reloading Modules, third-party Modules) are all out of v1.
- **Kohi's split** (a few large link-time libraries plus runtime-loaded plugins). Ember follows it for render backends and Applications, and like Kohi keeps the platform layer inside core.
