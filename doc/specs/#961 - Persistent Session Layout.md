---
author: John Stephens @johnstep
created on: 2021-04-03
last updated: 2021-04-14
issue id: 961
---

# Persistent Session Layout

## Abstract

This spec proposes a design for Windows Terminal (Terminal) to save its session
layout and automatically restore it when the session is restarted. It covers
the details of which state is saved and how, when it is saved, and when it is
restored, in addition to a new setting to disable automatic restoration. Future
work will include restoring pane buffers, command history, and undoing
accidental closing of panes, tabs, and windows.

## Solution Design

### Definitions

* *instance* - A single WindowsTerminal.exe process that (currently) manages
  exactly one main window with one or more tabs, each with one or more panes.
* *session* - All instances of WindowsTerminal.exe running in a Windows
  session, including elevated processes running as the same user. This does not
  include instances running as another user, elevated or not.

### Saved Layout State

Session state will be saved per Terminal instance by assigning a unique
identifier (GUID) to each new Terminal process on startup, unless it is
restoring state from a saved instance. The following state will be saved with
each instance:

1. Window position, state (maximized, minimized, restored), and virtual
   desktop, if any
2. Ordered list of tabs, the active tab, and for each tab:
    * The active pane
    * Name, if renamed
    * Color
3. Ordered list of panes per tab, and for each pane:
    * Profile
    * Split direction
    * Split size
4. Is elevated as the same user (a protected administrator)
    * **Issue:** Restarting elevated processes can be a confusing experience
      because each requires its own elevation prompt, and users might be
      confused by elevation prompts when signing in to Windows. This already
      happens, but it is worth considering solutions that avoid elevation
      prompts when signing in. One idea is to defer restarting any elevated
      instances until the first elevated instance is started, without any
      command-line parameters. Once elevated, it can restart any elevated
      instances without additional prompts. To make this work, elevated
      instances would not register for restart.
    * **Note:** If an instance is elevated as a *different* user, the state
      will be stored with that user and inaccessible by the current user. This
      will be most common for standard users, and must there elevate as a
      different user. The state will be stored with the other user. Terminal
      could detect this and run an instance as that user, resulting in an
      elevation prompt. That instance would then restore its saved windows. It
      is also possible to have Terminal windows running as a different user,
      but not elevated. That could be handled similarly by requesting
      credentials.

For the best restore experience, any other layout information missing from here
should be included, for example if the tab width can be changed. However, the
contents of any pane or customizations resulting from commands within a
console, such as setting a title from cmd.exe, are out of scope.

### Saving Layout State

Terminal will ideally update the saved layout state any time it changes, which
includes moving or sizing the window, minimizing or maximizing it, opening or
closing panes or tabs, changing tab names (directly from Terminal) or colors,
and sizing panes.

When a user closes a pane or tab, its state is immediately removed. When a user
closes a Terminal window, all of its state is immediately removed on WM_CLOSE.
However, when Windows closes a Terminal window due to a user exiting Windows,
or when the app is being updated, Terminal preserves the state by tracking this
on WM_ENDSESSION, ensuring it is not removed later even on WM_CLOSE, if
applicable.

Saving state as often as reasonable allows for up-to-date recovery even in
cases where the Terminal process crashes, Windows bug checks, the device loses
power, etc. Each Terminal instance will always save its state, which will only
be removed when the user closes a window. However, if the new Terminal restore
setting is off, then Terminal will *not* save the state when exiting Windows
unless Windows is going to restart apps when the user back signs in. Terminal
can detect the app restart case by the presence of the ENDSESSION_CLOSEAPP flag
in lParam on WM_ENDSESSION.

The state for Terminal instance, or window, can be saved in a package local
state directory, with one file per window (instance). Given the current process
architecture, this makes it easy to synchronize state access. A race to start
the same session more than once can be resolved by requesting exclusive write
access to the file; whichever process acquires it becomes the restored
instance, while any others gracefully terminate. The state will likely be saved
in JSON format, but is not meant to be edited directly by users. It should be
optimized for fast writing, especially or small changes, for example changing
the window position.

### Restoring Layout State

When the first non-elevated instance of Terminal *without any command-line
arguments* starts in a Windows session, it checks for any saved instances by
enumerated the save instance state files. It launches each saved instance, in
parallel, without ever creating its own window, and then exits. If an instance
fails to start, it is not removed and will be retried next time.

**Note:** Each time a new Terminal process is started without any command-line
arguments, it should attempt to first restart any remaining instances that had
failed to restart before, and not removing them if they fail again. However, it
should also create its own new window as it normally would. It might make sense
to remove an instance after repeated restart failures. Until a future design
adds undo, and potential some form of history, the user will have no easy way
to clean up any failed instances, and may not even know about them, so a policy
should be designed to remove them eventually.

There are different ways in which Terminal can be launched, including the
startup task (off by default), Windows app restart (depending on how Windows
was exited, and if the Windows app restart setting is on), and launching by the
user or other automated mechanisms. In order to provide a reasonable experience
when more than one of these end happen, only the *first* Terminal instance
launched with no command-line arguments (detected by creating and checking a
named runtime object) will attempt to launch the other saved instances. Each
Terminal instance will hold a reference to the named runtime object to prevent
any new Terminal instances from attempting to restart any saved instances.

Terminal will register for restart with *RegisterApplicationRestart* and supply
a per-instance restart command line in the form `--restore <GUID>` where the
GUID is the instance identifier. It will pass 0 for the flags parameter,
allowing it to be restarted if terminated due to a crash or hang, in addition
to exiting Windows or when Terminal is updated. Windows only allows restarting
after a crash or hang if the process had been running for at least a minute, to
avoid crash cycles.

When Windows restores the previous Terminal session through Windows app restart
or Restart Manager, Terminal will restore the window state as precisely as
possible, including minimization. When a *user* restores the session by
launching Terminal, it will not restore windows minimized even if they were.

## UI/UX Design

As defined in the Solution Design section, Terminal will automatically restore
the previous session when the user signs in or when the first instance is
launched. This design does not call for any prompts, but does define a new
setting, named 'Restore last session on first launch' that is on by default.

Users can turn off the setting to prevent Terminal from saving its state when
cleanly exiting Windows in the cases where Windows will not restart apps.
However, the state is still saved in other cases to handle crash recovery and
app restart when requested, for example unattended Windows Update restarts.

## Capabilities

### Accessibility

This might impact accessibility for users who always expect a single window
when starting, with their default profile loaded. The workaround would be to
turn off the new restoration setting.

### Security

Any data read by elevated Terminal instances must be processed with extra care.

### Reliability

This design should make Terminal more robust for users because they will not
lose their session every time their Terminal or Windows session is restarted,
whether for updating Terminal, Windows, or even after a crash or hang. This
design must provide a consistent, reliable experience.

### Compatibility

For users who currently use custom command lines to start instances of Terminal
with multiple tabs and/or panes, those should only be required when a user
intentionally closing them. Therefore, if a user has configured one or more
custom command lines to run automatically when signing in to Windows, it is
be best to undo that since Terminal will restore them automatically.

### Performance, Power, and Efficiency

Restoring many Terminal instances can take significant CPU and time.

## Potential Issues

* Some users might always want a clean slate after restarting. The new setting
  defined here addresses that when Windows is exited cleanly. But if Windows
  bug checks, the device loses power, etc. Terminal will always attempt to
  restore the previous session. The user can work around this by closing each
  restored window, or by simply exiting Windows and signing back in.

## Future considerations

* The design should take into consideration a possible move to a single UI
  process per [#5000 - Process Model 2.0].
* A future spec will propose a design for persistent pane buffers.
* A future spec will propose a design for persistent console state, which is
  primarily the command history per executable for a given pane.
* A future spec will propose a design to keep a history of closed panes, tabs,
  and windows, and allow users to restore them. This history may be limited to
  the current session, but should still be persisted for recovery.

## Resources

* Restore previously closed session's state [#961](https://github.com/microsoft/terminal/issues/961)

[#5000 - Process Model 2.0]: https://github.com/microsoft/terminal/blob/main/doc/specs/%235000%20-%20Process%20Model%202.0/%235000%20-%20Process%20Model%202.0.md
