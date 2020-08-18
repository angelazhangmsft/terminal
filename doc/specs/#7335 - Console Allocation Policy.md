---
author: Dustin Howett @DHowett <duhowett@microsoft.com>
created on: 2020-08-16
last updated: 2020-08-18
issue id: #7335
---

# Console Allocation Policy

## Abstract

Due to the design of the console subsystem on Windows as it has existed since Windows 95, every application that is
stamped with the `IMAGE_SUBSYSTEM_WINDOWS_CUI` subsystem in its PE header will be allocated a console by kernel32.

Any application that is stamped `IMAGE_SUBSYSTEM_WINDOWS_GUI` will not automatically be allocated a console.

This has worked fine for many years: when you double-click a console application in your GUI shell, it is allocated a
console.  When you run a GUI application from your console shell, it is **not** allocated a console. The shell will
**not** wait for it to exit before returning you to a prompt.

There is a large class of applications that has diverse console allocation needs. Take Python (or perhaps Perl, Ruby, or
Lua, or even our own VBScript). People author GUI applications in Python and Perl and VBScript, for better or for worse.

Because they are console subsystem applications, any user double-clicking a shortcut to a Python or Perl application
will be presented with a useless black box that the language runtime may choose to garbage collect, or may choose not
to.

Any user running that GUI application from a console shell will see their shell hang until the application terminates.

All of these scripting languages worked around this by shipping two binaries each, identical in every way expect in
their subsystem bits. python/pythonw, perl/perlw, ruby/rubyw, wscript/cscript.

Likewise, PowerShell\[1\] is waiting to deal with this problem because they don't necessarily want to ship a `pwshw.exe`
for all of their GUI-only authors.

On the other side, you have mostly-GUI applications that want to print output to a console **if there is one
connected**.  Sometimes they'll allocate their own (and therefore a new window) to display in, and sometimes they'll
reattach to the one they could have inherited. VSCode does the latter, and so when you run `code` from CMD, and then
`exit` CMD, your console window sticks around becuase VSCode is still attached to it. It will never print anything, so
you just have a dead console window.

There's a risk in reattaching, though. Given that the shell decides whether to wait based on the subsystem field, GUI
subsystem applications that reattach to their owning consoles *just to print some text* end up stomping on the output of
any shell that doesn't wait for them:

```
C:\> wt --help

wt - the Windows Terminal
C:\> Usage: [OPTIONS] ...
```

> _(the prompt is interleaved with the output)_

## Solution Design

I propose that we introduce a fusion manifest field, **consoleAllocationPolicy**, with the following values:

* `inheritOnly`
* `always`

It would look (roughly) like this:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
  <application>
    <windowsSettings>
      <consoleAllocationPolicy xmlns="http://schemas.microsoft.com/SMI/20XX/WindowsSettings">inheritOnly</activeCodePage>
    </windowsSettings>
  </application>
</assembly>
```

This field will apply consistent behavior across different image subsystems, with differing end user results:

|               | `SUBSYSTEM_GUI`                                                       | `SUBSYSTEM_GUI`               |
| -             | -                                                                     | -                             |
| _no entry_    | _default behavior_                                                    | _default behavior_            |
| `inheritOnly` | No console window unless application was started from console session | _default behavior_            |
| `always`      | _default behavior_                                                    | Always opens a console window |

An application that is in the *console* subsystem with a `consoleAllocationPolicy` of **inheritOnly** will not present a
console when launched from Explorer.

An application that is in the *windows* subsystem with a `consoleAllocationPolicy` of **always** *will always* be
allocated a new conhost on startup.

## Inspiration

Fusion manifest entries are used to make application-scoped decisions like this all the time, like `longPathAware` and
`heapType`.

CUI applications that can spawn a UI (or GUI applications that can print to a shell) are commonplace on other platforms,
because there is no subsystem differentiation.

## UI/UX Design

There is no UI for this feature.

## Capabilities

### Accessibility

This should have no impact on accessibility.

### Security

This should have no impact on security.

### Reliability

This should have no impact on reliability.

### Compatibility

On downlevel verisons of Windows that do not understand (or expect) this manifest field, applications will launch
consoles as specified by their image subsystem (described in the [abstract](#abstract) above).

This **will** constitute a breaking change for any application that opts into a change in behavior, but that breaking
change is expected to be managed by the application.

EXAMPLE: If Python migrates python.exe to specify an allocation policy of **inheritOnly**, graphical python applications
will become double-click runnable from the graphical shell without spawning a console window. _However_, console-based
python applications will no longer spawn a console window when double-clicked from the graphical shell.

Python could work around this by calling `AllocateConsole` if it can be detected that console I/O is required.

### Performance, Power, and Efficiency

This should have no impact on performance, power or efficiency.

## Potential Issues

I am **not** proposing a change in how shells determine whether to wait for an application before returning to a prompt.
This means that a console subsystem application that intends to primarily present a UI but occasionally print text to a
console (therefore choosing the **inheritOnly** allocation policy) will cause the shell to "hang" and wait for it to
exit.

Because the vast majority of shells on Windows "hang" by calling `WaitFor...Object` with a HANDLE to the spawned
process, an application that wants to be a "hybrid" CUI/GUI application will be forced to spawn a separate process to
detach from the shell and then terminate its main process.

This is very similar to the forking model seen in many POSIX-compliant operating systems.

## Future considerations

We're introducing a new manifest field today -- what if we want to introduce more? Should we have a `consoleSettings`
manifest block?

Are there other allocation policies we need to consider?

## Resources

### Rejected Solutions

- A new PE subsystem, `IMAGE_SUBSYSTEM_WINDOWS_HYBRID`
    - it would behave like **inheritOnly**
    - relies on shells to update and check for this
    - checking a subsystem doesn't work right with app execution aliases\[2\]
        - This is not a new problem, but it digs the hole a little deeper.
    - requires standardization outside of Microsoft because the PE format is a dependency of the UEFI specification\[3\]

- An exported symbol that shells can check for to determine whether to wait for the attached process to exit
    - relies on shells to update and check for this
    - cracking an executable to look for symbols is probably the last thing shells want to do
        - we could provide an API to determine whether to wait or return?
    - fragile, somewhat silly, exporting symbols from EXEs is obnoxious and uncommon

### Links

\[1\]: [Powershell -WindowStyle Hidden still shows a window briefly]
\[2\]: [PowerShell: Windows Store applications incorrectly assumed to be console applications]
\[3\]: [UEFI spec 2.6 appendix Q.1]

[Powershell -WindowStyle Hidden still shows a window briefly]: https://github.com/PowerShell/PowerShell/issues/3028
[PowerShell: Windows Store applications incorrectly assumed to be console applications]: https://github.com/PowerShell/PowerShell/issues/9970
[UEFI spec 2.6 appendix Q.1]: https://www.uefi.org/sites/default/files/resources/UEFI%20Spec%202_6.pdf