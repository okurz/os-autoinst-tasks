_Note:_ Not appearing numbers mean already done tasks, moved to
tasks/completed/

_Howto:_ For every numbered task create an according implementation planning
document in tasks/ in a numbered fashion, e.g. "11. my task" ->
"tasks/011_my_task.md" and inform the user. Do not execute the plan without
confirmation.

5 . *TBD*
9.  Create a detailed execution plan for the following request: in
    tasks/009_migrate_to_rust_python to migrate os-autoinst completely to Rust
    and Python away from any Perl or C/C++ dependency except for keeping a
    shallow Perl test-API. For this consider to replace the C++ and CMake
    controlled OpenCV/VNC component with Rust (Core), establish a Python
    bridge to this component, e.g. using PyO3, keep backward-compatible
    wrappers to allow to continue operations while the migration is ongoing.
    plan to migrate isotovideo, core components, backends, consoles,
    extensions to Rust+Python. Focus on fast, fully test covered, human
    readable, sustainable code and step wise migrations we can conduct in
    phases while keeping full functionality inact during every phase.
