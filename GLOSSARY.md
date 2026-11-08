# Ember

A modular 3D game engine written in C, targeting Windows, macOS and Linux desktop.

## Language

**Module**:
A piece of Ember built as its own dynamic library; a game includes only the modules it uses.
_Avoid_: subsystem, plugin, library

**Render backend**:
The implementation of Ember's renderer on one graphics API: Vulkan, Metal or Direct3D 12.
_Avoid_: driver, graphics backend

**Milestone**:
A stage of v1 that ends in something runnable; v1 is delivered as an ordered series of milestones.
_Avoid_: phase, sprint

**Reference game**:
The fantasy action-adventure RPG whose needs settle Ember's design trade-offs.
_Avoid_: sample game, demo game
