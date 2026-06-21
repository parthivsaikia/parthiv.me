---
title: "Hidden commands in Cobra: why they exist and where they're actually useful"
description: "Cobra's Hidden field looks pointless at first. Here's a real use case for it — spawning a detached background process to clear the clipboard after copying a password."
date: 2026-06-22
---

If you have used the cobra framework for building CLIs in Go, you might have come across the `Hidden` field in the `Command` struct. The docs describe it like this: _Hidden defines, if this command is hidden and should NOT show up in the list of available commands._

So the user won't see this command in your help menu or in autocomplete. But that doesn't mean they can't run it. If they know the command exists — say by reading your source code — they can call it just fine. `Hidden` only hides it from listing, it doesn't restrict access.

## Why would you even want this?

At first this feels pointless. Why have a command the user doesn't even know about? It's like a button on a website that's invisible but still clickable.

I had this exact reaction while learning cobra, reading this [primer on hidden commands](https://opdev.github.io/cobra-primer/hands_on/hidden_cmds.html). The example there is a command called `supersecretmath` that just prints a string — good for showing how the flag works, but it doesn't really explain why you'd want this in a real app.

I found a proper use case for it while building [enmasec](https://github.com/enmasec), a password manager.

## The problem: clearing the clipboard after copying a password

Like any decent password manager, enmasec can copy a password to the clipboard without printing it to the screen. The command looks like this:

```
enmasec account get [service] [account] --copy
```

But copying isn't the whole story. If the password just sits in the clipboard, anyone with access to the machine (or any process reading the clipboard) can grab it. So we need to clear it automatically after some time has passed.

In a TUI this is simple — the app is a long-running process, so you just schedule a `tea.Cmd` for later (still on my TODO list for enmasec's TUI). But a CLI is different. The moment `enmasec account get` finishes, the process exits. There's nothing left running to clear the clipboard a few seconds later.

So the flow we actually want is:

```
copy the password -> wait some time -> clear the clipboard
```

The process that does the copying can't be the one that waits and clears too, since it dies right after. It needs to spawn something else that survives after it exits. That's where `Hidden` comes in.

## Spawning a hidden command in the background

The idea: just before the main process exits, it spawns a new instance of itself, detached from the terminal, that runs a separate command whose only job is to wait and then clear the clipboard. This needs to be an internal command — not something a user should ever call on their own, since running it directly is a destructive action that should only be triggered by enmasec itself. So it has to be hidden.

Here's the hidden command:

```go
package clipboard

import (
	"io"
	"os"

	"github.com/spf13/cobra"
)

func ClearPasswordCmd() *cobra.Command {
	return &cobra.Command{
		Use:    "clear-password",
		Hidden: true,
		RunE: func(cmd *cobra.Command, args []string) error {
			// Read the password securely passed to us via stdin
			input, err := io.ReadAll(os.Stdin)
			if err != nil {
				return err
			}
			password := string(input)
			return ClearClipboard(password)
		},
	}
}
```

A couple of things changed here from the first version I had. The password isn't taken as a positional arg anymore — it's read straight off stdin inside the command itself, so it never has to pass through `args` at all. And `Run` became `RunE`, so any error from reading stdin or from `ClearClipboard` gets handled by Cobra the normal way, instead of being silently dropped and the process force-exited.

And `ClearClipboard` does the actual waiting and clearing:

```go
func ClearClipboard(content string) error {
	time.Sleep(30 * time.Second)
	if content == "" {
		return fmt.Errorf("can't clear empty string from clipboard")
	}
	clipboardContent, err := clipboard.ReadAll()
	if err != nil {
		return fmt.Errorf("can't read from clipboard: %w", err)
	}
	if content == clipboardContent {
		err = clipboard.WriteAll("")
		if err != nil {
			return err
		}
	}
	return nil
}
```

It checks that the clipboard still holds the same password before wiping it. That way, if the user copied something else in the meantime, we don't end up clearing that instead.

We register this on the root command:

```go
rootCmd.AddCommand(clipboard.ClearPasswordCmd())
```

Now `enmasec clear-password` exists as a real command, but it won't show up in `enmasec --help` or in shell autocomplete.

## Spawning it as a detached background process

The next piece is making `enmasec account get --copy` spawn this command in the background, detached from the parent process, so it keeps running after the parent exits.

```go
package clipboard

import (
	"io"
	"os"
	"os/exec"
)

func SpawnBackground(password string) error {
	exe, err := os.Executable()
	if err != nil {
		return err
	}
	cmd := exec.Command(exe, "clear-password")

	stdin, err := cmd.StdinPipe()
	if err != nil {
		return err
	}
	setSysProcAttr(cmd)
	if err := cmd.Start(); err != nil {
		return err
	}
	_, err = io.WriteString(stdin, password)
	_ = stdin.Close()
	return err
}
```

A few things worth calling out here:

- `os.Executable()` gives us the path to the currently running binary (enmasec itself), so we know exactly what to re-exec.
- We write the password into the child's stdin pipe, which matches up with `clear-password` reading it via `io.ReadAll(os.Stdin)` on the other end. A plain CLI argument would show up in `ps` output or shell history — stdin avoids that entirely.
- `setSysProcAttr` is what actually detaches the child process so it doesn't get killed along with the parent.

That last part is OS-specific, so it needs two separate implementations.

**For unix:**

```go
//go:build !windows

package clipboard

import (
	"os/exec"
	"syscall"
)

func setSysProcAttr(cmd *exec.Cmd) {
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Setsid: true,
	}
}
```

If the `Setsid` field is true, a new session is created and the child becomes the session leader. The child gets its own process group ID, so a `SIGINT` sent to the parent (like hitting Ctrl+C) can't kill this child anymore. It's fully detached from the parent.

**For windows:**

```go
//go:build windows

package clipboard

import (
	"os/exec"
	"syscall"
)

func setSysProcAttr(cmd *exec.Cmd) {
	cmd.SysProcAttr = &syscall.SysProcAttr{
		CreationFlags: 0x08000000,
	}
}
```

Windows doesn't have the `Setsid` concept, so it needs its own attribute instead — `0x08000000` is the `CREATE_NO_WINDOW` flag, which spawns the process without attaching a console window to the parent.

If you're wondering why there are two separate definitions for one function, it's because process attributes depend on the operating system. The `//go:build [os]` tag tells the Go compiler to strictly build that file only for the matching OS, so the right version gets picked automatically depending on where you compile.

## Putting it all together

With the hidden command registered and the spawner ready, the only thing left is to call `SpawnBackground` right where the password gets copied, inside the `enmasec account get` command:

```go
if err := clipboard.SpawnBackground(account.Password); err != nil {
		return "", err
}
```

That's it. `enmasec account get --copy` copies the password, spawns a detached `clear-password` instance in the background, and exits. 30 seconds later that background instance wakes up, checks if the clipboard still has the password, and wipes it if so.

## Takeaway

`Hidden` isn't access control or a security feature — it's purely a UI thing. It just keeps a command out of help text and autocomplete. But that's still genuinely useful when you need an internal command meant to be called only by your own binary, like re-spawning itself to do background work after the main process has already returned control to the shell.

## Wrapping up

The whole thing fits together in three small pieces: a hidden command that does the actual work, a function that spawns it detached from the parent, and a tiny bit of OS-specific glue to make "detached" actually mean detached. None of it is complicated on its own — the trick is just realizing that a process which is about to die can still leave something running behind it.

If you ever hit the "I need this to keep going after I exit" problem in a CLI, this is a pattern worth keeping in your back pocket.
