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
12. https://openqa.suse.de/tests/21504743#step/bootloader_zkvm/19 fails with
    "qemu-img create with backing file failed at
    /usr/lib/os-autoinst/consoles/sshVirtsh.pm line 472." .
    https://openqa.suse.de/tests/21504743/logfile?filename=autoinst-log.txt
    provides more details stating
```
s390x-34.3@s390x-kvm-minimal_with_BUILD34.3_xfstests.qcow2' 85899345920)] stderr:
  qemu-img: /var/lib/libvirt/images/openQA-SUT-14a.img: Failed to get "write" lock
  Is another process using the image [/var/lib/libvirt/images/openQA-SUT-14a.img]?
```
    This seems to keep happening mainly when multiple jobs with the same HDD
    asset try to run around the same time on the same worker host.
    We should improve the error reporting and at best prevent the error.
13. os-autoinst uses VNC based typing of commands and forwards a unique string
    together with the exit code of the command to a serial port e,g.
    /dev/ttyS0 . This works reliably for nearly 20 years now but was never
    pretty for test reviewers looking at the generated screenshots or videos.
    Also due to limited performance of typing over VNC this is slow and can
    even lead to problems under load of loosing characters  The command `fc`
    supports reading the history. So we should try to use this or another
    alternative to automatically forward a unique hash about the command we
    executed and its exit code to the serial port without needing to type it
    out over VNC.
