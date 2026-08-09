---
layout: post
title: "Is everything really a file?"
description: >-
  The slogan is true, and not in the way it sounds. Following it down explains why two handles can
  share one read position, and why the number you just closed comes straight back.
tags: [systems]
---

<p class="runbox"><b>Versions.</b> Source links are pinned: <code>torvalds/linux</code> at
<b>v7.1</b>, <code>apple-oss-distributions/xnu</code> at <b>xnu-11215.41.3</b>,
<code>golang/go</code> at <b>01534385</b>, <code>busybox</code> at <b>1_36_1</b>. Every program output
below is from <code>go1.25.12 linux/arm64</code> in a container rather than on bare metal, so the
numbers come from a virtual machine.</p>

**TL;DR** "Everything in Linux is a file" is the first thing anyone learns and the last thing anyone
checks. It is truer than it sounds: for things that are obviously not files, sockets included, the
kernel builds a real file object and attaches a table of functions saying what read and write mean
for it. That is why two handles can share one read position, and why a number you close comes
straight back. It is also why `cmd > file 2>&1` and `cmd 2>&1 > file` become two sequences you can
run in your head instead of two spellings you memorise.

## The slogan, and the first thing that does not fit

Open a file and look at what came back:

```python
# first.py
import os
print(os.open("/tmp/notes.txt", os.O_RDONLY))
```

```
$ echo hello > /tmp/notes.txt
$ python3 first.py
3
```

Three. Not an object, not a handle with methods, not a pointer. A number.

A number has to be a number *of* something, and you already know the first three entries of the
thing it counts. Descriptor 0 is stdin, 1 is stdout, 2 is stderr. Those are not special globals with
reserved names. They are the first three rows of a table your process carries, filled in before your
code ran, and `2>&1` is an instruction about that table: make row 2 point wherever row 1 points.

So `open()` did not hand you a file. It put one in the next free row and told you the row number.
Which is exactly why the number cannot be the file. Row 3 means nothing in another process, and
nothing to the kernel either without knowing who is asking.

<details class="tryit" markdown="1">
<summary>See the table for yourself</summary>

On Linux the table is readable as a directory. `exec 3<` opens a file into a slot you choose, and
`$$` is the shell's own pid:

```
$ echo hello > /tmp/notes.txt   # something worth opening
$ exec 3< /tmp/notes.txt        # open it for reading, into slot 3
$ ls -l /proc/$$/fd             # show me this shell's table

# trimmed: the permission, owner and date columns are cut, and the notes on
# the right are mine, not the shell's
0 -> /dev/pts/0        the terminal, as stdin
1 -> /dev/pts/0        the terminal, as stdout
2 -> /dev/pts/0        the terminal, as stderr
3 -> /tmp/notes.txt    the one we just opened
```

The middle line does three things at once. `< file` opens for reading, the `3` in front picks the
slot where a plain `< file` would have used 0, and `exec` with no command means "apply this to the
shell itself and keep it", because a redirection otherwise lasts for one command only.

No Linux machine needed:

```
docker run --rm -it alpine sh
```

Two things will show up that the argument above does not need.

**A descriptor you did not open.** There is usually a `10 -> /dev/tty`. That is the shell keeping a
private handle on the terminal for job control, duplicated to a high number so your redirections
cannot take it away. It appears only when the shell is interactive, which is what `-it` buys. The
reason is in the source:

```c
/* setjobctl in busybox shell/ash.c */
fd = fcntl(fd, F_DUPFD_CLOEXEC, 10);
...
pgrp = tcgetpgrp(fd);
if (pgrp < 0) {
 out:
  ash_msg("can't access tty; job control turned off");
```

[`setjobctl`](https://github.com/mirror/busybox/blob/1_36_1/shell/ash.c#L4077). The `10` means "lowest
free number, but not below 10", keeping the shell out of the 0 to 9 range that scripts assign by
hand. bash does the same thing at 255. Without that descriptor there is no `Ctrl+Z` and no `fg`.

**A different answer from `/proc/self/fd`.** That one is `ls`'s table rather than the shell's: it
inherited yours, then opened one more to read the directory, and that extra one is already gone by
the time its name would be printed.

macOS has no `/proc` at all. `lsof -p $$` prints the same table; read the `FD` column for the number
and `NAME` for what it points at.

</details>

That answers what `3` is not. Every strange thing in this post follows from what sits behind it.

## So what is behind the number?

Three levels, and nearly every surprise comes from collapsing two of them. This is not folklore.
The standard names the levels in the sentence that defines `open()`:

> It shall create an **open file description** that refers to a file and a **file descriptor** that
> refers to that open file description.

[POSIX.1-2024, `open()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/open.html)

Read it slowly, because it introduces two things whose names are nearly identical, and keeping them
apart is most of the work.

- **The file descriptor** is the number and nothing else. The standard defines it as "a per-process
  unique, non-negative integer".
- **The open file description** is the object that number refers to: "a record of how a process or
  group of processes is accessing a file". It holds the read and write position and the status flags.
- **The object underneath** is the level that sentence does not name. It is whatever the record is
  open on: an inode for a file on disk, a `struct socket` for a socket, a ring buffer for a pipe.

<figure class="diagram">
<svg viewBox="-3 -6 686 140" xmlns="http://www.w3.org/2000/svg" role="img"
     aria-label="Three levels. Level one is the per-process descriptor table, with slot 3 taken by the number open returned. Level two is the open file description, a struct file holding the offset, the status flags and the f_op table. Level three is the object underneath: an inode for a regular file, a struct socket for a socket, a ring buffer for a pipe. Only level one belongs to the process.">
  <style>
    .s   { font: 11.5px ui-sans-serif, system-ui, sans-serif; fill: #6b6b73; text-anchor: middle; }
    .hd  { font: 600 12px ui-sans-serif, system-ui, sans-serif; fill: #26262b; }
    .mono{ font: 11.5px ui-monospace, SFMono-Regular, Menlo, monospace; fill: #26262b; text-anchor: middle; }
    .box { fill: #ffffff; stroke: #b8b4ab; stroke-width: 1.4; }
    .on  { fill: #e8f2f0; stroke: #2f7d75; stroke-width: 1.6; }
    .ln  { stroke: #b8b4ab; stroke-width: 1.3; fill: none; }
  </style>

  <text class="hd" x="0"   y="10">your process</text>
  <text class="hd" x="246" y="10">kernel</text>

  <rect class="box" x="0" y="24" width="176" height="96" rx="4"/>
  <text class="s" x="88" y="42">1 · descriptor table</text>
  <rect class="box" x="16"  y="54" width="34" height="30" rx="3"/><text class="mono" x="33"  y="74">0</text>
  <rect class="box" x="56"  y="54" width="34" height="30" rx="3"/><text class="mono" x="73"  y="74">1</text>
  <rect class="box" x="96"  y="54" width="34" height="30" rx="3"/><text class="mono" x="113" y="74">2</text>
  <rect class="on"  x="136" y="54" width="34" height="30" rx="3"/><text class="mono" x="153" y="74">3</text>
  <text class="s" x="88" y="108">open() returned 3</text>

  <rect class="on" x="246" y="24" width="168" height="96" rx="4"/>
  <text class="s"    x="330" y="42">2 · open file description</text>
  <text class="mono" x="330" y="64">struct file</text>
  <text class="s"    x="330" y="86">offset, status flags</text>
  <text class="s"    x="330" y="106">f_op: read, write, close</text>

  <rect class="box" x="484" y="24" width="196" height="96" rx="4"/>
  <text class="s" x="582" y="42">3 · the object underneath</text>
  <text class="s" x="582" y="66">regular file → inode</text>
  <text class="s" x="582" y="86">socket → struct socket</text>
  <text class="s" x="582" y="106">pipe → ring buffer</text>

  <path class="ln" d="M176,69 L242,69" marker-end="url(#a2)"/>
  <path class="ln" d="M414,69 L480,69" marker-end="url(#a2)"/>
  <text class="s" x="209" y="60">fd_install</text>

  <defs>
    <marker id="a2" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto">
      <path d="M0,0 L7,3.5 L0,7 z" fill="#b8b4ab"/>
    </marker>
  </defs>
</svg>
<figcaption>The number on the left is yours. The other two levels are the kernel's.</figcaption>
</figure>

Which level a copy lands on turns out to be the whole game, and the next section is about that.

## The question that makes the middle box worth knowing

The middle box is invisible. You never allocate one, never name one, never see one in a debugger. So
why should you care that it is there?

Because of this situation, which you meet without going looking for it:

- a shell runs `2>&1`, and now stderr and stdout are the same thing. It is why
  `docker logs web 2>&1 | grep error` finds anything at all, and it is the answer half of Stack
  Overflow gives when a pipe comes back empty.
- a process opens a log file and forks, and now parent and child both have it.

In both cases two descriptors refer to one open file. **Do they share a read and write position, or
does each one keep its own?** The answer decides whether a forked child appending to a log corrupts
its parent's next write, and it is not something you can guess from the API.

The three-level model turns that into a question with one variable: **which level got copied?** Copy
at level 1 and both numbers reach one middle box, so there is one position between them. Call
`open()` twice and there are two middle boxes, so the positions are separate.

Which means it can be measured. `dup(fd)` copies at level 1, two `open()` calls do not, and the file
holds ten letters so a short read shows where each one starts:

```go
const path = "/tmp/fd-offset.txt"
must(os.WriteFile(path, []byte("ABCDEFGHIJ"), 0o644))

a := openRO(path) // syscall.Open(path, O_RDONLY, 0)
b, err := syscall.Dup(a)
must(err)
fmt.Printf("a := openRO(path)     a is fd %d\n", a)
fmt.Printf("b := dup(a)           b is fd %d\n", b)
fmt.Printf("read(a, 3)         -> %q\n", read(a, 3))
fmt.Printf("read(b, 3)         -> %q\n\n", read(b, 3))

c := openRO(path)
d := openRO(path)
fmt.Printf("c := openRO(path)     c is fd %d\n", c)
fmt.Printf("d := openRO(path)     d is fd %d\n", d)
fmt.Printf("read(c, 3)         -> %q\n", read(c, 3))
fmt.Printf("read(d, 3)         -> %q\n", read(d, 3))
```

```
a := openRO(path)     a is fd 3
b := dup(a)           b is fd 6
read(a, 3)         -> "ABC"
read(b, 3)         -> "DEF"

c := openRO(path)     c is fd 7
d := openRO(path)     d is fd 8
read(c, 3)         -> "ABC"
read(d, 3)         -> "ABC"
```

<figure class="steps">
<div class="steps-frames">
<svg viewBox="0 0 620 214" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 1. Slot 3 is taken and points at a new open file description with offset 0."><style>.sf-code{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sf-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sf-lblL{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sf-num{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sf-tbl{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-div{stroke:#e2ded6;stroke-width:1}.sf-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sf-hi{stroke:#2f7d75;stroke-width:1.3;fill:none}.sf-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sfa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="sfah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sf-code" x="16" y="20">a := open(path)</text><text class="sf-out" x="16" y="206">a is fd 3</text><text class="sf-lblL" x="16" y="48">descriptor table</text><rect class="sf-tbl" x="20" y="58" width="342" height="28" rx="3"/><line class="sf-div" x1="58" y1="59" x2="58" y2="85"/><line class="sf-div" x1="96" y1="59" x2="96" y2="85"/><line class="sf-div" x1="134" y1="59" x2="134" y2="85"/><line class="sf-div" x1="172" y1="59" x2="172" y2="85"/><line class="sf-div" x1="210" y1="59" x2="210" y2="85"/><line class="sf-div" x1="248" y1="59" x2="248" y2="85"/><line class="sf-div" x1="286" y1="59" x2="286" y2="85"/><line class="sf-div" x1="324" y1="59" x2="324" y2="85"/><rect class="sf-hot" x="134" y="58" width="38" height="28" rx="3"/><text class="sf-num" x="39" y="77" fill="#3a3a41">0</text><text class="sf-num" x="77" y="77" fill="#3a3a41">1</text><text class="sf-num" x="115" y="77" fill="#3a3a41">2</text><text class="sf-num" x="153" y="77" fill="#3a3a41">3</text><text class="sf-num" x="191" y="77" fill="#bab6ad">4</text><text class="sf-num" x="229" y="77" fill="#bab6ad">5</text><text class="sf-num" x="267" y="77" fill="#bab6ad">6</text><text class="sf-num" x="305" y="77" fill="#bab6ad">7</text><text class="sf-num" x="343" y="77" fill="#bab6ad">8</text><rect class="sf-boxon" x="70" y="132" width="190" height="52" rx="4"/><text class="sf-lbl" x="165" y="150">open file description</text><text class="sf-num" x="165" y="172" fill="#3a3a41">offset 0</text><path class="sf-hi" d="M153,88 L165,128" marker-end="url(#sfah)"/></svg>
<svg viewBox="0 0 620 214" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 2. Slot 6 is taken and points at the same open file description as slot 3."><style>.sf-code{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sf-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sf-lblL{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sf-num{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sf-tbl{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-div{stroke:#e2ded6;stroke-width:1}.sf-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sf-hi{stroke:#2f7d75;stroke-width:1.3;fill:none}.sf-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sfa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="sfah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sf-code" x="16" y="20">b := dup(a)</text><text class="sf-out" x="16" y="206">b is fd 6, pointing at the box a already had</text><text class="sf-lblL" x="16" y="48">descriptor table</text><rect class="sf-tbl" x="20" y="58" width="342" height="28" rx="3"/><line class="sf-div" x1="58" y1="59" x2="58" y2="85"/><line class="sf-div" x1="96" y1="59" x2="96" y2="85"/><line class="sf-div" x1="134" y1="59" x2="134" y2="85"/><line class="sf-div" x1="172" y1="59" x2="172" y2="85"/><line class="sf-div" x1="210" y1="59" x2="210" y2="85"/><line class="sf-div" x1="248" y1="59" x2="248" y2="85"/><line class="sf-div" x1="286" y1="59" x2="286" y2="85"/><line class="sf-div" x1="324" y1="59" x2="324" y2="85"/><rect class="sf-hot" x="248" y="58" width="38" height="28" rx="3"/><text class="sf-num" x="39" y="77" fill="#3a3a41">0</text><text class="sf-num" x="77" y="77" fill="#3a3a41">1</text><text class="sf-num" x="115" y="77" fill="#3a3a41">2</text><text class="sf-num" x="153" y="77" fill="#3a3a41">3</text><text class="sf-num" x="191" y="77" fill="#bab6ad">4</text><text class="sf-num" x="229" y="77" fill="#bab6ad">5</text><text class="sf-num" x="267" y="77" fill="#3a3a41">6</text><text class="sf-num" x="305" y="77" fill="#bab6ad">7</text><text class="sf-num" x="343" y="77" fill="#bab6ad">8</text><rect class="sf-boxon" x="70" y="132" width="190" height="52" rx="4"/><text class="sf-lbl" x="165" y="150">open file description</text><text class="sf-num" x="165" y="172" fill="#3a3a41">offset 0</text><path class="sf-ln" d="M153,88 L165,128" marker-end="url(#sfa)"/><path class="sf-hi" d="M267,88 L165,128" marker-end="url(#sfah)"/></svg>
<svg viewBox="0 0 620 214" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 3. Reading through slot 3 returns ABC and moves the shared offset to 3."><style>.sf-code{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sf-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sf-lblL{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sf-num{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sf-tbl{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-div{stroke:#e2ded6;stroke-width:1}.sf-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sf-hi{stroke:#2f7d75;stroke-width:1.3;fill:none}.sf-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sfa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="sfah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sf-code" x="16" y="20">read(a, 3)</text><text class="sf-out" x="16" y="206">&quot;ABC&quot;   the offset moved, inside the box</text><text class="sf-lblL" x="16" y="48">descriptor table</text><rect class="sf-tbl" x="20" y="58" width="342" height="28" rx="3"/><line class="sf-div" x1="58" y1="59" x2="58" y2="85"/><line class="sf-div" x1="96" y1="59" x2="96" y2="85"/><line class="sf-div" x1="134" y1="59" x2="134" y2="85"/><line class="sf-div" x1="172" y1="59" x2="172" y2="85"/><line class="sf-div" x1="210" y1="59" x2="210" y2="85"/><line class="sf-div" x1="248" y1="59" x2="248" y2="85"/><line class="sf-div" x1="286" y1="59" x2="286" y2="85"/><line class="sf-div" x1="324" y1="59" x2="324" y2="85"/><text class="sf-num" x="39" y="77" fill="#3a3a41">0</text><text class="sf-num" x="77" y="77" fill="#3a3a41">1</text><text class="sf-num" x="115" y="77" fill="#3a3a41">2</text><text class="sf-num" x="153" y="77" fill="#3a3a41">3</text><text class="sf-num" x="191" y="77" fill="#bab6ad">4</text><text class="sf-num" x="229" y="77" fill="#bab6ad">5</text><text class="sf-num" x="267" y="77" fill="#3a3a41">6</text><text class="sf-num" x="305" y="77" fill="#bab6ad">7</text><text class="sf-num" x="343" y="77" fill="#bab6ad">8</text><rect class="sf-boxon" x="70" y="132" width="190" height="52" rx="4"/><text class="sf-lbl" x="165" y="150">open file description</text><text class="sf-num" x="165" y="172" fill="#3a3a41">offset 3</text><path class="sf-hi" d="M153,88 L165,128" marker-end="url(#sfah)"/><path class="sf-ln" d="M267,88 L165,128" marker-end="url(#sfa)"/></svg>
<svg viewBox="0 0 620 214" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 4. Reading through slot 6 returns DEF, because it reads the same offset."><style>.sf-code{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sf-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sf-lblL{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sf-num{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sf-tbl{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-div{stroke:#e2ded6;stroke-width:1}.sf-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sf-hi{stroke:#2f7d75;stroke-width:1.3;fill:none}.sf-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sfa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="sfah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sf-code" x="16" y="20">read(b, 3)</text><text class="sf-out" x="16" y="206">&quot;DEF&quot;   b never touched an offset, yet b sees it</text><text class="sf-lblL" x="16" y="48">descriptor table</text><rect class="sf-tbl" x="20" y="58" width="342" height="28" rx="3"/><line class="sf-div" x1="58" y1="59" x2="58" y2="85"/><line class="sf-div" x1="96" y1="59" x2="96" y2="85"/><line class="sf-div" x1="134" y1="59" x2="134" y2="85"/><line class="sf-div" x1="172" y1="59" x2="172" y2="85"/><line class="sf-div" x1="210" y1="59" x2="210" y2="85"/><line class="sf-div" x1="248" y1="59" x2="248" y2="85"/><line class="sf-div" x1="286" y1="59" x2="286" y2="85"/><line class="sf-div" x1="324" y1="59" x2="324" y2="85"/><text class="sf-num" x="39" y="77" fill="#3a3a41">0</text><text class="sf-num" x="77" y="77" fill="#3a3a41">1</text><text class="sf-num" x="115" y="77" fill="#3a3a41">2</text><text class="sf-num" x="153" y="77" fill="#3a3a41">3</text><text class="sf-num" x="191" y="77" fill="#bab6ad">4</text><text class="sf-num" x="229" y="77" fill="#bab6ad">5</text><text class="sf-num" x="267" y="77" fill="#3a3a41">6</text><text class="sf-num" x="305" y="77" fill="#bab6ad">7</text><text class="sf-num" x="343" y="77" fill="#bab6ad">8</text><rect class="sf-boxon" x="70" y="132" width="190" height="52" rx="4"/><text class="sf-lbl" x="165" y="150">open file description</text><text class="sf-num" x="165" y="172" fill="#3a3a41">offset 6</text><path class="sf-ln" d="M153,88 L165,128" marker-end="url(#sfa)"/><path class="sf-hi" d="M267,88 L165,128" marker-end="url(#sfah)"/></svg>
<svg viewBox="0 0 620 214" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 5. Two more opens take slots 7 and 8, each with its own description at offset 0."><style>.sf-code{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sf-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sf-lblL{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sf-num{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sf-tbl{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-div{stroke:#e2ded6;stroke-width:1}.sf-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sf-hi{stroke:#2f7d75;stroke-width:1.3;fill:none}.sf-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sfa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="sfah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sf-code" x="16" y="20">c, d := open(path), open(path)</text><text class="sf-out" x="16" y="206">c is fd 7, d is fd 8, each with a box of its own</text><text class="sf-lblL" x="16" y="48">descriptor table</text><rect class="sf-tbl" x="20" y="58" width="342" height="28" rx="3"/><line class="sf-div" x1="58" y1="59" x2="58" y2="85"/><line class="sf-div" x1="96" y1="59" x2="96" y2="85"/><line class="sf-div" x1="134" y1="59" x2="134" y2="85"/><line class="sf-div" x1="172" y1="59" x2="172" y2="85"/><line class="sf-div" x1="210" y1="59" x2="210" y2="85"/><line class="sf-div" x1="248" y1="59" x2="248" y2="85"/><line class="sf-div" x1="286" y1="59" x2="286" y2="85"/><line class="sf-div" x1="324" y1="59" x2="324" y2="85"/><rect class="sf-hot" x="324" y="58" width="38" height="28" rx="3"/><rect class="sf-hot" x="286" y="58" width="38" height="28" rx="3"/><text class="sf-num" x="39" y="77" fill="#3a3a41">0</text><text class="sf-num" x="77" y="77" fill="#3a3a41">1</text><text class="sf-num" x="115" y="77" fill="#3a3a41">2</text><text class="sf-num" x="153" y="77" fill="#3a3a41">3</text><text class="sf-num" x="191" y="77" fill="#bab6ad">4</text><text class="sf-num" x="229" y="77" fill="#bab6ad">5</text><text class="sf-num" x="267" y="77" fill="#3a3a41">6</text><text class="sf-num" x="305" y="77" fill="#3a3a41">7</text><text class="sf-num" x="343" y="77" fill="#3a3a41">8</text><rect class="sf-box" x="70" y="132" width="190" height="52" rx="4"/><text class="sf-lbl" x="165" y="150">open file description</text><text class="sf-num" x="165" y="172" fill="#3a3a41">offset 6</text><rect class="sf-boxon" x="300" y="132" width="140" height="52" rx="4"/><text class="sf-lbl" x="370" y="150">open file description</text><text class="sf-num" x="370" y="172" fill="#3a3a41">offset 0</text><rect class="sf-boxon" x="462" y="132" width="140" height="52" rx="4"/><text class="sf-lbl" x="532" y="150">open file description</text><text class="sf-num" x="532" y="172" fill="#3a3a41">offset 0</text><path class="sf-ln" d="M153,88 L165,128" marker-end="url(#sfa)"/><path class="sf-ln" d="M267,88 L165,128" marker-end="url(#sfa)"/><path class="sf-hi" d="M305,88 L370,128" marker-end="url(#sfah)"/><path class="sf-hi" d="M343,88 L532,128" marker-end="url(#sfah)"/></svg>
<svg viewBox="0 0 620 214" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 6. Both new descriptors read ABC, because their offsets are independent."><style>.sf-code{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sf-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sf-lblL{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sf-num{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sf-tbl{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-div{stroke:#e2ded6;stroke-width:1}.sf-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sf-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.3}.sf-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sf-hi{stroke:#2f7d75;stroke-width:1.3;fill:none}.sf-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sfa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="sfah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sf-code" x="16" y="20">read(c, 3);  read(d, 3)</text><text class="sf-out" x="16" y="206">&quot;ABC&quot;  &quot;ABC&quot;   two boxes, two positions</text><text class="sf-lblL" x="16" y="48">descriptor table</text><rect class="sf-tbl" x="20" y="58" width="342" height="28" rx="3"/><line class="sf-div" x1="58" y1="59" x2="58" y2="85"/><line class="sf-div" x1="96" y1="59" x2="96" y2="85"/><line class="sf-div" x1="134" y1="59" x2="134" y2="85"/><line class="sf-div" x1="172" y1="59" x2="172" y2="85"/><line class="sf-div" x1="210" y1="59" x2="210" y2="85"/><line class="sf-div" x1="248" y1="59" x2="248" y2="85"/><line class="sf-div" x1="286" y1="59" x2="286" y2="85"/><line class="sf-div" x1="324" y1="59" x2="324" y2="85"/><text class="sf-num" x="39" y="77" fill="#3a3a41">0</text><text class="sf-num" x="77" y="77" fill="#3a3a41">1</text><text class="sf-num" x="115" y="77" fill="#3a3a41">2</text><text class="sf-num" x="153" y="77" fill="#3a3a41">3</text><text class="sf-num" x="191" y="77" fill="#bab6ad">4</text><text class="sf-num" x="229" y="77" fill="#bab6ad">5</text><text class="sf-num" x="267" y="77" fill="#3a3a41">6</text><text class="sf-num" x="305" y="77" fill="#3a3a41">7</text><text class="sf-num" x="343" y="77" fill="#3a3a41">8</text><rect class="sf-box" x="70" y="132" width="190" height="52" rx="4"/><text class="sf-lbl" x="165" y="150">open file description</text><text class="sf-num" x="165" y="172" fill="#3a3a41">offset 6</text><rect class="sf-boxon" x="300" y="132" width="140" height="52" rx="4"/><text class="sf-lbl" x="370" y="150">open file description</text><text class="sf-num" x="370" y="172" fill="#3a3a41">offset 3</text><rect class="sf-boxon" x="462" y="132" width="140" height="52" rx="4"/><text class="sf-lbl" x="532" y="150">open file description</text><text class="sf-num" x="532" y="172" fill="#3a3a41">offset 3</text><path class="sf-ln" d="M153,88 L165,128" marker-end="url(#sfa)"/><path class="sf-ln" d="M267,88 L165,128" marker-end="url(#sfa)"/><path class="sf-hi" d="M305,88 L370,128" marker-end="url(#sfah)"/><path class="sf-hi" d="M343,88 L532,128" marker-end="url(#sfah)"/></svg>
</div>
<figcaption>The offset lives in the middle box, never in the number.</figcaption>
</figure>

Nobody moved a position on `b`, because `b` is a number and numbers do not have positions. The
position is one level down, and both numbers reach it. Two `open` calls build two of those instead,
so the same bytes at level 3 sit under two positions at level 2 that know nothing of each other.

<details class="tryit" markdown="1">
<summary>Why the numbers are 3, 6, 7, 8 and not 3, 4, 5, 6</summary>

Nothing in that program took 4 or 5, so something else did. The program below is `probe`, and its
whole job is to list what the process is already holding when `main` reaches its first line:

```go
func listFDs(label string) {
  fmt.Printf("--- %s ---\n", label)
  entries, err := os.ReadDir("/proc/self/fd")
  if err != nil {
    fmt.Println("  no /proc (not Linux):", err)
    return
  }
  for _, e := range entries {
    target, _ := os.Readlink(filepath.Join("/proc/self/fd", e.Name()))
    fmt.Printf("  fd %-3s -> %s\n", e.Name(), target)
  }
}
```

```
--- at the top of main, before the three opens below ---
  fd 0   -> /dev/null
  fd 1   -> pipe:[51163285]
  fd 2   -> pipe:[51163286]
  fd 3   -> 
  fd 4   -> anon_inode:[eventpoll]
  fd 5   -> anon_inode:[eventfd]
```

`fd 4` and `fd 5` are the network poller: an epoll instance and an eventfd to interrupt it. Which
raises a better question than it looks. The rule is lowest free first, and 3 was free, so why did
the poller start at 4?

```
$ strace -e trace=openat,epoll_create1,eventfd2,epoll_ctl,close ./probe   # the program above

openat(AT_FDCWD, "/tmp/fd-probe.txt", O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC, 0644) = 3
epoll_create1(EPOLL_CLOEXEC)            = 4
eventfd2(0, EFD_CLOEXEC|EFD_NONBLOCK)   = 5
epoll_ctl(4, EPOLL_CTL_ADD, 5, {events=EPOLLIN, ...}) = 0
epoll_ctl(4, EPOLL_CTL_ADD, 3, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, ...}) = -1 EPERM
close(3)                                = 0
openat(AT_FDCWD, "/proc/self/fd", O_RDONLY|O_CLOEXEC|O_DIRECTORY) = 3
```

fd 3 was not free. The poller is built the first time the program hands a descriptor to the
runtime, which is the `os.WriteFile` on the first line of `main`, still open when `epoll_create1`
runs. So 4 and 5 were the lowest numbers available and the rule held. The rest of the trace
explains the listing itself: `close(3)` gives the number back, the directory handle doing the
listing takes it, and it is closed again before its own name can be printed, which is the blank
`fd 3`. One line there matters later: `epoll_ctl` on the regular file is refused with `EPERM`, and
that refusal is what the last section of this post is about.

</details>

That is the answer to the question this section opened with. A forked child reading from an inherited
descriptor moves the parent's position, because `fork` copies the table and not the boxes. Two
independent `open()` calls on one log file do not interfere, because each built its own box.

<details class="tryit" markdown="1">
<summary>Why <code>2&gt;&amp;1</code> is the same operation as <code>dup</code></summary>

The syntax reads like aiming one stream at another. The standard files it under *Duplicating an
Output File Descriptor*: `[n]>&word` "shall duplicate one output file descriptor from another"
([POSIX.1-2024, Redirection](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/V3_chap02.html#tag_19_07_06)).
In the shell it is one call:

```c
/* redirect in busybox shell/ash.c, running "2>&1" */
fd    = redir->nfile.fd;     /* N, left of the >     2 */
...
newfd = redir->ndup.dupfd;   /* M, after the &       1 */
...
dup2_or_raise(newfd, fd);    /* copy slot 1 into slot 2 */
```

[`redirect`](https://github.com/mirror/busybox/blob/1_36_1/shell/ash.c#L5827) ·
[`dup2_or_raise`](https://github.com/mirror/busybox/blob/1_36_1/shell/ash.c#L5625)

So `2>&1` inherits the result above: stdout and stderr end up sharing one position, because they
share one middle box.

</details>

<details class="tryit" markdown="1">
<summary>Ruling out the other explanation, in the kernel</summary>

One story was left standing. Instead of two slots reaching one box, each slot could get **its own**
box, with both boxes pointing down at one level 3 object. The experiment already rules that out, a
shared position needs a shared box, but it is worth seeing that the kernel says the same thing.

`dup2(oldfd, newfd)` enters the kernel as `ksys_dup3`, which is the implementation both `dup2` and
`dup3` are built on
([`SYSCALL_DEFINE2(dup2)`](https://github.com/torvalds/linux/blob/v7.1/fs/file.c#L1464)). Three
identifiers carry the whole thing, and each one is a box from the diagram:

| In the code | In the diagram |
|---|---|
| `files` / `fdt` | level 1, this process's descriptor table |
| `file` | level 2, the open file description |
| `fdt->fd[fd]` | one slot of level 1, holding a pointer to level 2 |

```c
/* ksys_dup3, then do_dup2 which it calls, in fs/file.c */
file = files_lookup_fd_locked(files, oldfd);   /* read the level 2 pointer out of the old slot */
...
get_file(file);                                /* say one more slot is about to hold it        */
rcu_assign_pointer(fdt->fd[fd], file);         /* write that same pointer into the new slot    */
```

[`ksys_dup3`](https://github.com/torvalds/linux/blob/v7.1/fs/file.c#L1425) ·
[`do_dup2`](https://github.com/torvalds/linux/blob/v7.1/fs/file.c#L1294)

Read the last two lines as `table[newfd] = table[oldfd]`. `get_file` does not read anything; it
raises the reference count, which is how level 2 knows how many slots point at it. Nothing is
allocated, so there is no second box to be had, which is what POSIX requires when it says `dup2`
"shall cause the file descriptor `fildes2` to refer to **the same open file description**"
([POSIX.1-2024, `dup`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/dup.html)).

</details>

## Now try it on something that is definitely not a file

A file on disk fitting the file abstraction proves nothing. A socket is the real test. It has no
bytes at rest, no length, no position you can seek to, and it did not exist a second ago.

So ask the kernel the same three questions about each of them, and compare the answers:

- **the name**, the one procfs shows for it
- **the type**, what `fstat` says it is
- **the reopen**, whether that name can be opened

Two results look plausible before running it. If a socket sits outside the file world, `fstat`
refuses it and there is no inode to report. If it sits fully inside, all three answers come back
looking like the file's. Neither is what happens, and the question they split on is the reopen.

`report` below asks those three questions of whatever descriptor it is given, so it can be pointed
at a regular file and a TCP socket in turn and the answers compared side by side. It reads `/proc`,
so this one is Linux only too.

```go
func report(label string, fd int) {
  path := fmt.Sprintf("/proc/self/fd/%d", fd)
  name, _ := os.Readlink(path) // 1. what is it called
  kind, ino := fstatKind(fd)   // 2. what is it

  fmt.Printf("%s: fd %d\n", label, fd)
  fmt.Printf("  the name procfs shows   %s\n", name)
  fmt.Printf("  fstat says it is        %s, inode %d\n", kind, ino)

  reopened, err := syscall.Open(path, syscall.O_RDONLY, 0) // 3. open that name
  if err != nil {
    fmt.Printf("  open() that name        %v\n", err)
  } else {
    fmt.Printf("  open() that name        worked, got fd %d\n", reopened)
    _ = syscall.Close(reopened)
  }
  fmt.Println()
}

const path = "/tmp/fd-regular.txt"
must(os.WriteFile(path, []byte("hello"), 0o644))
rf, err := syscall.Open(path, syscall.O_RDONLY, 0)
must(err)
report("regular file", rf)

sk, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0)
must(err)
report("tcp socket", sk)
```

```
regular file: fd 4
  the name procfs shows   /tmp/fd-regular.txt
  fstat says it is        regular file, inode 116947266
  open() that name        worked, got fd 7

tcp socket: fd 7
  the name procfs shows   socket:[53454226]
  fstat says it is        socket, inode 53454226
  open() that name        no such device or address
```

<figure class="steps">
<div class="steps-frames">
<svg viewBox="0 0 620 152" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 1. Open takes slot 4, the lowest one free."><style>.sr-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sr-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sr-s{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sr-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sr-on{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.5}.sr-free{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:3 3}.sr-n{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41;text-anchor:middle}.sr-nd{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#bab6ad;text-anchor:middle}.sr-a{font:11px ui-sans-serif,system-ui,sans-serif;fill:#2f7d75;text-anchor:middle}.sr-am{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}</style><text class="sr-c" x="20" y="22">rf, _ := syscall.Open("/tmp/fd-regular.txt", ...)</text><text class="sr-h" x="20" y="46">your descriptor table</text><text class="sr-h" x="600" y="46" text-anchor="end">grey: 0-2 stdio, 3 and 5-6 held by the Go runtime</text><rect class="sr-sys" x="24" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="53" y="79">0</text><rect class="sr-sys" x="86" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="115" y="79">1</text><rect class="sr-sys" x="148" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="177" y="79">2</text><rect class="sr-sys" x="210" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="239" y="79">3</text><rect class="sr-on" x="272" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="301" y="79">4</text><rect class="sr-sys" x="334" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="363" y="79">5</text><rect class="sr-sys" x="396" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="425" y="79">6</text><rect class="sr-free" x="458" y="56" width="58" height="36" rx="3"/><text class="sr-nd" x="487" y="79">7</text><rect class="sr-free" x="520" y="56" width="58" height="36" rx="3"/><text class="sr-nd" x="549" y="79">8</text><text class="sr-a" x="301" y="112">the file, at the lowest free slot</text></svg>
<svg viewBox="0 0 620 152" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 2. Inside report, opening the procfs name takes slot 7."><style>.sr-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sr-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sr-s{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sr-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sr-on{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.5}.sr-free{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:3 3}.sr-n{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41;text-anchor:middle}.sr-nd{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#bab6ad;text-anchor:middle}.sr-a{font:11px ui-sans-serif,system-ui,sans-serif;fill:#2f7d75;text-anchor:middle}.sr-am{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}</style><text class="sr-c" x="20" y="22">report(rf)   ->  syscall.Open("/proc/self/fd/4", ...)</text><text class="sr-h" x="20" y="46">your descriptor table</text><text class="sr-h" x="600" y="46" text-anchor="end">grey: 0-2 stdio, 3 and 5-6 held by the Go runtime</text><rect class="sr-sys" x="24" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="53" y="79">0</text><rect class="sr-sys" x="86" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="115" y="79">1</text><rect class="sr-sys" x="148" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="177" y="79">2</text><rect class="sr-sys" x="210" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="239" y="79">3</text><rect class="sr-s" x="272" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="301" y="79">4</text><rect class="sr-sys" x="334" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="363" y="79">5</text><rect class="sr-sys" x="396" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="425" y="79">6</text><rect class="sr-on" x="458" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="487" y="79">7</text><rect class="sr-free" x="520" y="56" width="58" height="36" rx="3"/><text class="sr-nd" x="549" y="79">8</text><text class="sr-a" x="487" y="112">the same file again, a second number</text></svg>
<svg viewBox="0 0 620 152" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 3. Report closes its copy and slot 7 is free again."><style>.sr-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sr-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sr-s{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sr-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sr-on{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.5}.sr-free{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:3 3}.sr-n{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41;text-anchor:middle}.sr-nd{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#bab6ad;text-anchor:middle}.sr-a{font:11px ui-sans-serif,system-ui,sans-serif;fill:#2f7d75;text-anchor:middle}.sr-am{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}</style><text class="sr-c" x="20" y="22">report(rf)   returns, closing the copy it opened</text><text class="sr-h" x="20" y="46">your descriptor table</text><text class="sr-h" x="600" y="46" text-anchor="end">grey: 0-2 stdio, 3 and 5-6 held by the Go runtime</text><rect class="sr-sys" x="24" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="53" y="79">0</text><rect class="sr-sys" x="86" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="115" y="79">1</text><rect class="sr-sys" x="148" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="177" y="79">2</text><rect class="sr-sys" x="210" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="239" y="79">3</text><rect class="sr-s" x="272" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="301" y="79">4</text><rect class="sr-sys" x="334" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="363" y="79">5</text><rect class="sr-sys" x="396" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="425" y="79">6</text><rect class="sr-free" x="458" y="56" width="58" height="36" rx="3"/><text class="sr-nd" x="487" y="79">7</text><rect class="sr-free" x="520" y="56" width="58" height="36" rx="3"/><text class="sr-nd" x="549" y="79">8</text><text class="sr-am" x="487" y="112">7 is free again</text></svg>
<svg viewBox="0 0 620 152" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 4. Socket takes slot 7, the number just released."><style>.sr-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sr-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sr-s{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sr-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sr-on{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.5}.sr-free{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:3 3}.sr-n{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41;text-anchor:middle}.sr-nd{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#bab6ad;text-anchor:middle}.sr-a{font:11px ui-sans-serif,system-ui,sans-serif;fill:#2f7d75;text-anchor:middle}.sr-am{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}</style><text class="sr-c" x="20" y="22">sk, _ := syscall.Socket(syscall.AF_INET, ...)</text><text class="sr-h" x="20" y="46">your descriptor table</text><text class="sr-h" x="600" y="46" text-anchor="end">grey: 0-2 stdio, 3 and 5-6 held by the Go runtime</text><rect class="sr-sys" x="24" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="53" y="79">0</text><rect class="sr-sys" x="86" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="115" y="79">1</text><rect class="sr-sys" x="148" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="177" y="79">2</text><rect class="sr-sys" x="210" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="239" y="79">3</text><rect class="sr-s" x="272" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="301" y="79">4</text><rect class="sr-sys" x="334" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="363" y="79">5</text><rect class="sr-sys" x="396" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="425" y="79">6</text><rect class="sr-on" x="458" y="56" width="58" height="36" rx="3"/><text class="sr-n" x="487" y="79">7</text><rect class="sr-free" x="520" y="56" width="58" height="36" rx="3"/><text class="sr-nd" x="549" y="79">8</text><text class="sr-a" x="487" y="112">the socket, at the number just released</text></svg>
</div>
<figcaption>Sockets and files compete for the same numbers, lowest free slot first.</figcaption>
</figure>

Two answers agree and one splits, and the split is the finding. That `fstat` answers at all, with a
real inode rather than a zero, puts the socket inside the file world. The same number inside
`socket:[...]` makes the procfs name the object itself, not a label for it. What the socket does not
have is a path that reaches it. So the line between these two was never whether they are files. It
is whether a name can be opened.

None of that is read off the outside. The call that builds a socket's file hands it a real inode, a
filesystem the kernel mounts for itself, and a table giving `read`, `write` and `close` a meaning
for a socket. That is the whole of "everything is a file". Not a metaphor and not a fake.

<details class="tryit" markdown="1">
<summary>Where the kernel says so, in <code>sock_alloc_file</code></summary>

```c
/* sock_alloc_file in net/socket.c */
file = alloc_file_pseudo(SOCK_INODE(sock), sock_mnt, dname,
                        O_RDWR | (flags & O_NONBLOCK),
                        &socket_file_ops);
...
file->private_data = sock;
```

[`sock_alloc_file`](https://github.com/torvalds/linux/blob/v7.1/net/socket.c#L536)

Three of those arguments carry the whole claim, and the rest are flags:

| In the call | What it is |
|---|---|
| `SOCK_INODE(sock)` | a real inode, the one `fstat` just reported |
| `sock_mnt` | the filesystem it sits on: sockfs, which the kernel [mounts for itself](https://github.com/torvalds/linux/blob/v7.1/net/socket.c#L3398) and never puts in your tree |
| `&socket_file_ops` | the table that gives `read`, `write` and `close` a meaning for a socket |

In C, `file->private_data` is Go's `file.private_data`. The arrow is what C writes when `file` is a
pointer. So the last line reads: in the file object, set the field `private_data` to this socket.
That field is declared `void *private_data` and documented as "filesystem or driver specific data",
a slot with no fixed type where each kind of file keeps whatever it actually is. Here it keeps the
socket, so the file object knows what it is a file of.

</details>

<details class="tryit" markdown="1">
<summary>What <code>socket()</code> does with that file</summary>

Three steps: take a free number, build the file object, then point the number at it. Only the middle
step knows what a socket is.

```c
/* sock_map_fd in net/socket.c */
int fd = get_unused_fd_flags(flags);   // the lowest free slot
...
newfile = sock_alloc_file(sock, flags, NULL);
if (!IS_ERR(newfile)) {
  fd_install(fd, newfile);
  return fd;
}
```

[`sock_map_fd`](https://github.com/torvalds/linux/blob/v7.1/net/socket.c#L564)

The number you get back from `socket()` is allocated by the same `get_unused_fd_flags` that `open()`
uses, which is why sockets and files compete for the same numbers in one process.

</details>

## How long it lives

<p class="gloss">One rule sets the lifetime, and the rest is what counts as a reference.</p>
It lives until nothing refers to it. That is the whole rule, and everything surprising about
lifetimes comes from what counts.

A name counts. Closing a descriptor on a disk file does not delete the file, because the directory
entry still names it, and that entry is a reference in its own right. The file outlives every
process that had it open.

A socket has no name, so the descriptors are the entire count, and your table slot is the last thing
holding it up. When the process exits the kernel closes every descriptor it held, each close is an
`fput`, and the final one runs:

```c
/* __fput in fs/file_table.c */
if (file->f_op->release)
  file->f_op->release(inode, file);
```

[`__fput`](https://github.com/torvalds/linux/blob/v7.1/fs/file_table.c#L484)

For a socket that `release` is
[`sock_close`](https://github.com/torvalds/linux/blob/v7.1/net/socket.c#L157-L168), one entry in the
same `socket_file_ops` table `sock_alloc_file` attached earlier. It calls `__sock_release`:

```c
/* __sock_release in net/socket.c */
static void __sock_release(struct socket *sock, struct inode *inode)
{
  const struct proto_ops *ops = READ_ONCE(sock->ops);   // the protocol's table, below f_op

  if (ops) {
    struct module *owner = ops->owner;

    if (inode)
      inode_lock(inode);      // two closers must not race
    ops->release(sock);       // the real close: connection down, buffers freed
    sock->sk = NULL;          // drop the protocol's control block
    if (inode)
      inode_unlock(inode);
    sock->ops = NULL;
    module_put(owner);
  }

  ...
  if (!sock->file) {
    iput(SOCK_INODE(sock));   // no file wrapped this socket, so drop the inode here
    return;
  }
  WRITE_ONCE(sock->file, NULL);  // the file releases the inode itself, in __fput
}
```

[`__sock_release`](https://github.com/torvalds/linux/blob/v7.1/net/socket.c#L713-L738)

The one thing the code cannot show is which function `ops->release` is. TCP and UDP share
`inet_release`, which then calls `sk->sk_prot->close`; a Unix socket takes `unix_release`. So the
file layer never learns what closing a connection means. It calls one pointer, and the protocol
below it does the work.

## Can anything outside your process hold it?

If the file has no usable name, your process looks like the only thing that could reach it. It is
not, and the two exceptions are worth knowing because they are how the reference count outlives you.

**`fork()`.** The child gets the table, not copies of the objects in it
([`fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html)). Same middle box, two tables
pointing at it. The socket now survives either process on its own.

<details class="tryit" markdown="1">
<summary>Two processes, one read position</summary>

Two questions. If the parent has already read part of a file, does the child start from that
position or from zero? And once both are running, is there one position between them or two?

Both need two live processes, so the program re-runs itself: the parent opens the file, reads from
it, then starts a copy of its own binary and hands the descriptor over. The pipes are only there to
take turns, so the two reads cannot overlap and leave the answer ambiguous.

```go
const path = "/tmp/fd-fork.txt"

func read(fd, n int) string {
  buf := make([]byte, n)
  m, _ := syscall.Read(fd, buf)
  return string(buf[:m])
}

func signal(fd int) { syscall.Write(fd, []byte{1}) }

func wait(fd int)   { syscall.Read(fd, make([]byte, 1)) }

if os.Getenv("FD_CHILD") == "1" {
  // 3 is the file, 4 is wait-for-parent, 5 is tell-parent
  fmt.Printf("child  read(3, 3)  -> %q\n", read(3, 3)) // read is syscall.Read
  signal(5)
  wait(4)
  fmt.Printf("child  read(3, 3)  -> %q\n", read(3, 3))
  signal(5)
  return
}

os.WriteFile(path, []byte("ABCDEFGHIJKL"), 0o644)
f, _ := os.Open(path)
fd := int(f.Fd())

fromParent, toChild, _ := os.Pipe() // parent writes toChild, child reads fromParent
fromChild, toParent, _ := os.Pipe() // child writes toParent, parent reads fromChild

fmt.Printf("parent read(%d, 3)  -> %q\n", fd, read(fd, 3))

child := exec.Command(os.Args[0]) // the fork is in here: clone, then execve
child.Env = append(os.Environ(), "FD_CHILD=1")
child.Stdout, child.Stderr = os.Stdout, os.Stderr
child.ExtraFiles = []*os.File{f, fromParent, toParent} // dup3'd into the child as 3, 4, 5
child.Start()

wait(int(fromChild.Fd()))
fmt.Printf("parent read(%d, 3)  -> %q\n", fd, read(fd, 3))
signal(int(toChild.Fd()))

wait(int(fromChild.Fd()))
child.Wait()
```

```
parent read(4, 3)  -> "ABC"
child  read(3, 3)  -> "DEF"
parent read(4, 3)  -> "GHI"
child  read(3, 3)  -> "JKL"
```

<figure class="steps">
<div class="steps-frames">
<svg viewBox="0 0 620 236" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 1. The parent opens the file. Slot 4 in its table points at a new open file description with the offset at 0. No child yet."><style>.sp-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sp-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sp-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sp-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sp-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sp-cell{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sp-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sp-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sp-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="spa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="spah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sp-c" x="16" y="20">f, _ := os.Open(path)</text><text class="sp-h" x="20" y="42">parent</text><rect class="sp-sys" x="20" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="110" y="50" width="30" height="28" rx="3"/><rect class="sp-hot" x="140" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="50" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="69" fill="#3a3a41">0</text><text class="sp-n" x="65" y="69" fill="#3a3a41">1</text><text class="sp-n" x="95" y="69" fill="#3a3a41">2</text><text class="sp-n" x="125" y="69" fill="#bab6ad">3</text><text class="sp-n" x="155" y="69" fill="#3a3a41">4</text><text class="sp-n" x="185" y="69" fill="#bab6ad">5</text><text class="sp-n" x="215" y="69" fill="#bab6ad">6</text><text class="sp-n" x="245" y="69" fill="#bab6ad">7</text><text class="sp-n" x="275" y="69" fill="#bab6ad">8</text><text class="sp-h" x="20" y="162">child, not started yet</text><rect class="sp-off" x="20" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="50" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="80" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="110" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="140" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="170" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="189" fill="#bab6ad">0</text><text class="sp-n" x="65" y="189" fill="#bab6ad">1</text><text class="sp-n" x="95" y="189" fill="#bab6ad">2</text><text class="sp-n" x="125" y="189" fill="#bab6ad">3</text><text class="sp-n" x="155" y="189" fill="#bab6ad">4</text><text class="sp-n" x="185" y="189" fill="#bab6ad">5</text><text class="sp-n" x="215" y="189" fill="#bab6ad">6</text><text class="sp-n" x="245" y="189" fill="#bab6ad">7</text><text class="sp-n" x="275" y="189" fill="#bab6ad">8</text><rect class="sp-boxon" x="370" y="86" width="200" height="56" rx="4"/><text class="sp-lbl" x="470" y="108">open file description</text><text class="sp-n" x="470" y="130" fill="#3a3a41">offset 0</text><path class="sp-hi" d="M155,78 L366,104" marker-end="url(#spah)"/><text class="sp-out" x="16" y="222">the parent gets fd 4, and a box with the position at 0</text></svg>
<svg viewBox="0 0 620 236" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 2. The parent reads three bytes and gets ABC. The offset in the box moves to 3."><style>.sp-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sp-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sp-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sp-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sp-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sp-cell{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sp-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sp-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sp-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="spa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="spah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sp-c" x="16" y="20">parent read(4, 3)</text><text class="sp-h" x="20" y="42">parent</text><rect class="sp-sys" x="20" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="110" y="50" width="30" height="28" rx="3"/><rect class="sp-hot" x="140" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="50" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="69" fill="#3a3a41">0</text><text class="sp-n" x="65" y="69" fill="#3a3a41">1</text><text class="sp-n" x="95" y="69" fill="#3a3a41">2</text><text class="sp-n" x="125" y="69" fill="#bab6ad">3</text><text class="sp-n" x="155" y="69" fill="#3a3a41">4</text><text class="sp-n" x="185" y="69" fill="#bab6ad">5</text><text class="sp-n" x="215" y="69" fill="#bab6ad">6</text><text class="sp-n" x="245" y="69" fill="#bab6ad">7</text><text class="sp-n" x="275" y="69" fill="#bab6ad">8</text><text class="sp-h" x="20" y="162">child, not started yet</text><rect class="sp-off" x="20" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="50" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="80" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="110" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="140" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="170" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="189" fill="#bab6ad">0</text><text class="sp-n" x="65" y="189" fill="#bab6ad">1</text><text class="sp-n" x="95" y="189" fill="#bab6ad">2</text><text class="sp-n" x="125" y="189" fill="#bab6ad">3</text><text class="sp-n" x="155" y="189" fill="#bab6ad">4</text><text class="sp-n" x="185" y="189" fill="#bab6ad">5</text><text class="sp-n" x="215" y="189" fill="#bab6ad">6</text><text class="sp-n" x="245" y="189" fill="#bab6ad">7</text><text class="sp-n" x="275" y="189" fill="#bab6ad">8</text><rect class="sp-boxon" x="370" y="86" width="200" height="56" rx="4"/><text class="sp-lbl" x="470" y="108">open file description</text><text class="sp-n" x="470" y="130" fill="#3a3a41">offset 3</text><path class="sp-hi" d="M155,78 L366,104" marker-end="url(#spah)"/><text class="sp-out" x="16" y="222">&quot;ABC&quot;   the position moves, inside the box</text></svg>
<svg viewBox="0 0 620 236" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 3. The child starts. Slot 3 in the child table points at the same open file description, whose offset is still 3."><style>.sp-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sp-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sp-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sp-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sp-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sp-cell{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sp-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sp-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sp-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="spa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="spah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sp-c" x="16" y="20">child.Start()</text><text class="sp-h" x="20" y="42">parent</text><rect class="sp-sys" x="20" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="110" y="50" width="30" height="28" rx="3"/><rect class="sp-hot" x="140" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="50" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="69" fill="#3a3a41">0</text><text class="sp-n" x="65" y="69" fill="#3a3a41">1</text><text class="sp-n" x="95" y="69" fill="#3a3a41">2</text><text class="sp-n" x="125" y="69" fill="#bab6ad">3</text><text class="sp-n" x="155" y="69" fill="#3a3a41">4</text><text class="sp-n" x="185" y="69" fill="#bab6ad">5</text><text class="sp-n" x="215" y="69" fill="#bab6ad">6</text><text class="sp-n" x="245" y="69" fill="#bab6ad">7</text><text class="sp-n" x="275" y="69" fill="#bab6ad">8</text><text class="sp-h" x="20" y="162">child</text><rect class="sp-sys" x="20" y="170" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="170" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="170" width="30" height="28" rx="3"/><rect class="sp-hot" x="110" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="140" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="170" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="189" fill="#3a3a41">0</text><text class="sp-n" x="65" y="189" fill="#3a3a41">1</text><text class="sp-n" x="95" y="189" fill="#3a3a41">2</text><text class="sp-n" x="125" y="189" fill="#3a3a41">3</text><text class="sp-n" x="155" y="189" fill="#bab6ad">4</text><text class="sp-n" x="185" y="189" fill="#bab6ad">5</text><text class="sp-n" x="215" y="189" fill="#bab6ad">6</text><text class="sp-n" x="245" y="189" fill="#bab6ad">7</text><text class="sp-n" x="275" y="189" fill="#bab6ad">8</text><rect class="sp-boxon" x="370" y="86" width="200" height="56" rx="4"/><text class="sp-lbl" x="470" y="108">open file description</text><text class="sp-n" x="470" y="130" fill="#3a3a41">offset 3</text><path class="sp-ln" d="M155,78 L366,104" marker-end="url(#spa)"/><path class="sp-hi" d="M125,170 L366,126" marker-end="url(#spah)"/><text class="sp-out" x="16" y="222">the child starts at 3, not at 0</text></svg>
<svg viewBox="0 0 620 236" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 4. The child reads three bytes and gets DEF, carrying on from the parent position. The offset moves to 6."><style>.sp-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sp-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sp-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sp-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sp-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sp-cell{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sp-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sp-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sp-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="spa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="spah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sp-c" x="16" y="20">child  read(3, 3)</text><text class="sp-h" x="20" y="42">parent</text><rect class="sp-sys" x="20" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="110" y="50" width="30" height="28" rx="3"/><rect class="sp-hot" x="140" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="50" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="69" fill="#3a3a41">0</text><text class="sp-n" x="65" y="69" fill="#3a3a41">1</text><text class="sp-n" x="95" y="69" fill="#3a3a41">2</text><text class="sp-n" x="125" y="69" fill="#bab6ad">3</text><text class="sp-n" x="155" y="69" fill="#3a3a41">4</text><text class="sp-n" x="185" y="69" fill="#bab6ad">5</text><text class="sp-n" x="215" y="69" fill="#bab6ad">6</text><text class="sp-n" x="245" y="69" fill="#bab6ad">7</text><text class="sp-n" x="275" y="69" fill="#bab6ad">8</text><text class="sp-h" x="20" y="162">child</text><rect class="sp-sys" x="20" y="170" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="170" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="170" width="30" height="28" rx="3"/><rect class="sp-hot" x="110" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="140" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="170" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="189" fill="#3a3a41">0</text><text class="sp-n" x="65" y="189" fill="#3a3a41">1</text><text class="sp-n" x="95" y="189" fill="#3a3a41">2</text><text class="sp-n" x="125" y="189" fill="#3a3a41">3</text><text class="sp-n" x="155" y="189" fill="#bab6ad">4</text><text class="sp-n" x="185" y="189" fill="#bab6ad">5</text><text class="sp-n" x="215" y="189" fill="#bab6ad">6</text><text class="sp-n" x="245" y="189" fill="#bab6ad">7</text><text class="sp-n" x="275" y="189" fill="#bab6ad">8</text><rect class="sp-boxon" x="370" y="86" width="200" height="56" rx="4"/><text class="sp-lbl" x="470" y="108">open file description</text><text class="sp-n" x="470" y="130" fill="#3a3a41">offset 6</text><path class="sp-ln" d="M155,78 L366,104" marker-end="url(#spa)"/><path class="sp-hi" d="M125,170 L366,126" marker-end="url(#spah)"/><text class="sp-out" x="16" y="222">&quot;DEF&quot;   it carries on from where the parent stopped</text></svg>
<svg viewBox="0 0 620 236" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 5. The parent reads and gets GHI, three letters further on than where it stopped, because the child moved the shared position."><style>.sp-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sp-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sp-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sp-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sp-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sp-cell{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sp-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sp-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sp-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="spa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="spah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sp-c" x="16" y="20">parent read(4, 3)</text><text class="sp-h" x="20" y="42">parent</text><rect class="sp-sys" x="20" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="110" y="50" width="30" height="28" rx="3"/><rect class="sp-hot" x="140" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="50" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="69" fill="#3a3a41">0</text><text class="sp-n" x="65" y="69" fill="#3a3a41">1</text><text class="sp-n" x="95" y="69" fill="#3a3a41">2</text><text class="sp-n" x="125" y="69" fill="#bab6ad">3</text><text class="sp-n" x="155" y="69" fill="#3a3a41">4</text><text class="sp-n" x="185" y="69" fill="#bab6ad">5</text><text class="sp-n" x="215" y="69" fill="#bab6ad">6</text><text class="sp-n" x="245" y="69" fill="#bab6ad">7</text><text class="sp-n" x="275" y="69" fill="#bab6ad">8</text><text class="sp-h" x="20" y="162">child</text><rect class="sp-sys" x="20" y="170" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="170" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="170" width="30" height="28" rx="3"/><rect class="sp-hot" x="110" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="140" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="170" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="189" fill="#3a3a41">0</text><text class="sp-n" x="65" y="189" fill="#3a3a41">1</text><text class="sp-n" x="95" y="189" fill="#3a3a41">2</text><text class="sp-n" x="125" y="189" fill="#3a3a41">3</text><text class="sp-n" x="155" y="189" fill="#bab6ad">4</text><text class="sp-n" x="185" y="189" fill="#bab6ad">5</text><text class="sp-n" x="215" y="189" fill="#bab6ad">6</text><text class="sp-n" x="245" y="189" fill="#bab6ad">7</text><text class="sp-n" x="275" y="189" fill="#bab6ad">8</text><rect class="sp-boxon" x="370" y="86" width="200" height="56" rx="4"/><text class="sp-lbl" x="470" y="108">open file description</text><text class="sp-n" x="470" y="130" fill="#3a3a41">offset 9</text><path class="sp-hi" d="M155,78 L366,104" marker-end="url(#spah)"/><path class="sp-ln" d="M125,170 L366,126" marker-end="url(#spa)"/><text class="sp-out" x="16" y="222">&quot;GHI&quot;   the child moved the parent&#39;s position</text></svg>
<svg viewBox="0 0 620 236" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 6. The child reads again and gets JKL, continuing from where the parent stopped. The two processes are taking turns on one position."><style>.sp-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sp-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sp-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sp-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sp-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sp-cell{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sp-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sp-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sp-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sp-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sp-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="spa" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="spah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sp-c" x="16" y="20">child  read(3, 3)</text><text class="sp-h" x="20" y="42">parent</text><rect class="sp-sys" x="20" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="50" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="110" y="50" width="30" height="28" rx="3"/><rect class="sp-hot" x="140" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="50" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="50" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="69" fill="#3a3a41">0</text><text class="sp-n" x="65" y="69" fill="#3a3a41">1</text><text class="sp-n" x="95" y="69" fill="#3a3a41">2</text><text class="sp-n" x="125" y="69" fill="#bab6ad">3</text><text class="sp-n" x="155" y="69" fill="#3a3a41">4</text><text class="sp-n" x="185" y="69" fill="#bab6ad">5</text><text class="sp-n" x="215" y="69" fill="#bab6ad">6</text><text class="sp-n" x="245" y="69" fill="#bab6ad">7</text><text class="sp-n" x="275" y="69" fill="#bab6ad">8</text><text class="sp-h" x="20" y="162">child</text><rect class="sp-sys" x="20" y="170" width="30" height="28" rx="3"/><rect class="sp-sys" x="50" y="170" width="30" height="28" rx="3"/><rect class="sp-sys" x="80" y="170" width="30" height="28" rx="3"/><rect class="sp-hot" x="110" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="140" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="170" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="200" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="230" y="170" width="30" height="28" rx="3"/><rect class="sp-off" x="260" y="170" width="30" height="28" rx="3"/><text class="sp-n" x="35" y="189" fill="#3a3a41">0</text><text class="sp-n" x="65" y="189" fill="#3a3a41">1</text><text class="sp-n" x="95" y="189" fill="#3a3a41">2</text><text class="sp-n" x="125" y="189" fill="#3a3a41">3</text><text class="sp-n" x="155" y="189" fill="#bab6ad">4</text><text class="sp-n" x="185" y="189" fill="#bab6ad">5</text><text class="sp-n" x="215" y="189" fill="#bab6ad">6</text><text class="sp-n" x="245" y="189" fill="#bab6ad">7</text><text class="sp-n" x="275" y="189" fill="#bab6ad">8</text><rect class="sp-boxon" x="370" y="86" width="200" height="56" rx="4"/><text class="sp-lbl" x="470" y="108">open file description</text><text class="sp-n" x="470" y="130" fill="#3a3a41">offset 12</text><path class="sp-ln" d="M155,78 L366,104" marker-end="url(#spa)"/><path class="sp-hi" d="M125,170 L366,126" marker-end="url(#spah)"/><text class="sp-out" x="16" y="222">&quot;JKL&quot;   and back again, still one position</text></svg>
</div>
<figcaption>Two tables, two different numbers, one position between them.</figcaption>
</figure>


The position crossed as it stood rather than starting again at zero, and after that neither side
ever rewinds or repeats the other. Two copies of the offset would have printed `ABC`, `ABC`, `DEF`,
`DEF`.

The two numbers differ and it changes nothing: each table numbered the slot its own way, and both
slots hold one box.

Now the three lines that carry the experiment, because none of them says `fork`.

`exec.Command(...).Start()` is the fork. Go will not fork on its own, so the runtime clones and then
execs, which the trace shows as `clone(...|CLONE_VFORK|SIGCHLD)` followed by `execve`. What survives
that is the descriptor table, which is the half of `fork` these two questions are about.

`ExtraFiles` is needed because Go opens every file close-on-exec,
`openat("/tmp/fd-fork.txt", O_RDONLY|O_CLOEXEC)`, so the descriptor would be gone the moment the
child execs. Handing it over is one `dup3(4, 3, 0)` in the child before the exec: slot 4 of the
parent copied into slot 3 of the child, with the flag cleared. That is `dup2`, and the `2>&1` toggle
already settled what `dup2` does. So the sharing is not something `ExtraFiles` invents. A real
`fork()` brings every descriptor across without being asked; here three are asked for by name.

The two pipes carry one byte each and do nothing but order the reads. Without them the interleaving
would be a race, and a racy demo proves nothing.

</details>

**Passing it over a Unix socket.** The one that surprised me: the same sharing, between processes
with no relationship at all. The man page for
[`SCM_RIGHTS`](https://man7.org/linux/man-pages/man7/unix.7.html) is careful about the wording,
because "passing a file descriptor" describes it badly. The number does not travel. The receiver
gets a number of its own, out of its own table, pointing at the box the sender is using. A file with
no usable name crosses a process boundary, and what crosses is a reference.

One machine only. `AF_UNIX` is for processes on the same host, and the limit is the mechanism rather
than an omission: what crosses is a reference to an object in one kernel. Another machine has
another kernel, and nothing to point at.

<details class="tryit" markdown="1">
<summary>The same result, between strangers</summary>

Same three reads as above, except nothing forked anything. One program built as `passfd` and
started twice from a shell: once as the receiver, which waits on a socket in `/tmp`, and once as
the sender, which owns the file. That socket is the only thing the two have in common.

```go
const (
  path = "/tmp/fd-scm.txt"
  sock = "/tmp/fd-scm.sock"
)

func sendFD(c *net.UnixConn, fd int) {
  c.WriteMsgUnix([]byte{0}, syscall.UnixRights(fd), nil)
}

func recvFD(c *net.UnixConn) int {
  oob := make([]byte, syscall.CmsgSpace(4))
  _, oobn, _, _, _ := c.ReadMsgUnix(make([]byte, 1), oob)
  scms, _ := syscall.ParseSocketControlMessage(oob[:oobn])
  fds, _ := syscall.ParseUnixRights(&scms[0])
  return fds[0]
}

func dial(path string) *net.UnixConn {
  addr := &net.UnixAddr{Name: path, Net: "unix"}
  for i := 0; i < 200; i++ {
    if c, err := net.DialUnix("unix", nil, addr); err == nil {
      return c
    }
    time.Sleep(5 * time.Millisecond)
  }
  panic("no receiver listening on " + path)
}

func send() {
  os.WriteFile(path, []byte("ABCDEFGHI"), 0o644)
  f, _ := os.Open(path)
  fd := int(f.Fd())
  fmt.Printf("sender   read(%d, 3)  -> %q\n", fd, read(fd, 3)) // read is syscall.Read

  c := dial(sock) // the same path receive() listens on
  sendFD(c, fd)

  c.Read(make([]byte, 1)) // wait until the receiver has read
  fmt.Printf("sender   read(%d, 3)  -> %q\n", fd, read(fd, 3))
}

func receive() {
  l, _ := net.ListenUnix("unix", &net.UnixAddr{Name: sock, Net: "unix"})

  c, _ := l.AcceptUnix()
  fd := recvFD(c)
  fmt.Printf("receiver read(%d, 3)  -> %q\n", fd, read(fd, 3))
  c.Write([]byte{0})
}

func main() {
  if len(os.Args) > 1 && os.Args[1] == "send" {
    send()
    return
  }
  receive()
}
```

```
$ ./passfd receive &
$ ./passfd send

sender   read(4, 3)  -> "ABC"
receiver read(8, 3)  -> "DEF"
sender   read(4, 3)  -> "GHI"
```

<figure class="steps">
<div class="steps-frames">
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 1. The sender opens the file and holds it at slot 4, pointing at an open file description with the offset at 0. The receiver has its own table with a listening socket and a connection, and nothing pointing at the file."><style>.sm-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sm-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sm-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sm-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sm-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sm-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sm-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sm-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-pill{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:4 3}.sm-pillon{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.sm-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.sm-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sm-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sm-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sma" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="smah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sm-c" x="16" y="20">sender: f, _ := os.Open(path)</text><text class="sm-h" x="20" y="44">sender, its own table</text><rect class="sm-sys" x="20" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="110" y="52" width="30" height="28" rx="3"/><rect class="sm-hot" x="140" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="170" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="200" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="230" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="52" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="71" fill="#3a3a41">0</text><text class="sm-n" x="65" y="71" fill="#3a3a41">1</text><text class="sm-n" x="95" y="71" fill="#3a3a41">2</text><text class="sm-n" x="125" y="71" fill="#3a3a41">3</text><text class="sm-n" x="155" y="71" fill="#3a3a41">4</text><text class="sm-n" x="185" y="71" fill="#bab6ad">5</text><text class="sm-n" x="215" y="71" fill="#bab6ad">6</text><text class="sm-n" x="245" y="71" fill="#bab6ad">7</text><text class="sm-n" x="275" y="71" fill="#bab6ad">8</text><text class="sm-h" x="20" y="172">receiver, a table of its own</text><rect class="sm-sys" x="20" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="110" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="140" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="170" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="200" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="230" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="180" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="199" fill="#3a3a41">0</text><text class="sm-n" x="65" y="199" fill="#3a3a41">1</text><text class="sm-n" x="95" y="199" fill="#3a3a41">2</text><text class="sm-n" x="125" y="199" fill="#bab6ad">3</text><text class="sm-n" x="155" y="199" fill="#3a3a41">4</text><text class="sm-n" x="185" y="199" fill="#3a3a41">5</text><text class="sm-n" x="215" y="199" fill="#3a3a41">6</text><text class="sm-n" x="245" y="199" fill="#3a3a41">7</text><text class="sm-n" x="275" y="199" fill="#bab6ad">8</text><rect class="sm-boxon" x="380" y="102" width="200" height="56" rx="4"/><text class="sm-lbl" x="480" y="124">open file description</text><text class="sm-n" x="480" y="146" fill="#3a3a41">offset 0</text><path class="sm-hi" d="M155,80 L376,120" marker-end="url(#smah)"/><text class="sm-out" x="16" y="236">the sender holds fd 4; the receiver has never seen this file</text></svg>
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 2. The sender reads three bytes and gets ABC. The offset in the box moves to 3."><style>.sm-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sm-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sm-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sm-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sm-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sm-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sm-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sm-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-pill{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:4 3}.sm-pillon{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.sm-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.sm-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sm-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sm-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sma" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="smah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sm-c" x="16" y="20">sender read(4, 3)</text><text class="sm-h" x="20" y="44">sender, its own table</text><rect class="sm-sys" x="20" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="110" y="52" width="30" height="28" rx="3"/><rect class="sm-hot" x="140" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="170" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="200" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="230" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="52" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="71" fill="#3a3a41">0</text><text class="sm-n" x="65" y="71" fill="#3a3a41">1</text><text class="sm-n" x="95" y="71" fill="#3a3a41">2</text><text class="sm-n" x="125" y="71" fill="#3a3a41">3</text><text class="sm-n" x="155" y="71" fill="#3a3a41">4</text><text class="sm-n" x="185" y="71" fill="#bab6ad">5</text><text class="sm-n" x="215" y="71" fill="#bab6ad">6</text><text class="sm-n" x="245" y="71" fill="#bab6ad">7</text><text class="sm-n" x="275" y="71" fill="#bab6ad">8</text><text class="sm-h" x="20" y="172">receiver, a table of its own</text><rect class="sm-sys" x="20" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="110" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="140" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="170" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="200" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="230" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="180" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="199" fill="#3a3a41">0</text><text class="sm-n" x="65" y="199" fill="#3a3a41">1</text><text class="sm-n" x="95" y="199" fill="#3a3a41">2</text><text class="sm-n" x="125" y="199" fill="#bab6ad">3</text><text class="sm-n" x="155" y="199" fill="#3a3a41">4</text><text class="sm-n" x="185" y="199" fill="#3a3a41">5</text><text class="sm-n" x="215" y="199" fill="#3a3a41">6</text><text class="sm-n" x="245" y="199" fill="#3a3a41">7</text><text class="sm-n" x="275" y="199" fill="#bab6ad">8</text><rect class="sm-boxon" x="380" y="102" width="200" height="56" rx="4"/><text class="sm-lbl" x="480" y="124">open file description</text><text class="sm-n" x="480" y="146" fill="#3a3a41">offset 3</text><path class="sm-hi" d="M155,80 L376,120" marker-end="url(#smah)"/><text class="sm-out" x="16" y="236">&quot;ABC&quot;   the position moves to 3</text></svg>
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 3. The sender puts the descriptor in the out-of-band part of a message. Traced on the way out it reads SCM_RIGHTS with cmsg_data of 4."><style>.sm-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sm-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sm-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sm-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sm-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sm-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sm-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sm-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-pill{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:4 3}.sm-pillon{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.sm-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.sm-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sm-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sm-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sma" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="smah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sm-c" x="16" y="20">sendFD(c, fd)</text><text class="sm-h" x="20" y="44">sender, its own table</text><rect class="sm-sys" x="20" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="110" y="52" width="30" height="28" rx="3"/><rect class="sm-hot" x="140" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="170" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="200" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="230" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="52" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="71" fill="#3a3a41">0</text><text class="sm-n" x="65" y="71" fill="#3a3a41">1</text><text class="sm-n" x="95" y="71" fill="#3a3a41">2</text><text class="sm-n" x="125" y="71" fill="#3a3a41">3</text><text class="sm-n" x="155" y="71" fill="#3a3a41">4</text><text class="sm-n" x="185" y="71" fill="#bab6ad">5</text><text class="sm-n" x="215" y="71" fill="#bab6ad">6</text><text class="sm-n" x="245" y="71" fill="#bab6ad">7</text><text class="sm-n" x="275" y="71" fill="#bab6ad">8</text><text class="sm-h" x="20" y="172">receiver, a table of its own</text><rect class="sm-sys" x="20" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="110" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="140" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="170" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="200" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="230" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="180" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="199" fill="#3a3a41">0</text><text class="sm-n" x="65" y="199" fill="#3a3a41">1</text><text class="sm-n" x="95" y="199" fill="#3a3a41">2</text><text class="sm-n" x="125" y="199" fill="#bab6ad">3</text><text class="sm-n" x="155" y="199" fill="#3a3a41">4</text><text class="sm-n" x="185" y="199" fill="#3a3a41">5</text><text class="sm-n" x="215" y="199" fill="#3a3a41">6</text><text class="sm-n" x="245" y="199" fill="#3a3a41">7</text><text class="sm-n" x="275" y="199" fill="#bab6ad">8</text><rect class="sm-pillon" x="55" y="112" width="200" height="36" rx="18"/><text class="sm-pt" x="155" y="135">SCM_RIGHTS  [4]</text><rect class="sm-boxon" x="380" y="102" width="200" height="56" rx="4"/><text class="sm-lbl" x="480" y="124">open file description</text><text class="sm-n" x="480" y="146" fill="#3a3a41">offset 3</text><path class="sm-hi" d="M155,80 L376,120" marker-end="url(#smah)"/><text class="sm-out" x="16" y="236">the message leaves carrying the sender&#39;s number</text></svg>
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 4. The same message arrives. The number in it now reads 8, because the receiving kernel took the lowest free slot in the receiver table and wrote that in. Slot 8 points at the sender box, whose offset is still 3."><style>.sm-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sm-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sm-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sm-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sm-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sm-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sm-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sm-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-pill{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:4 3}.sm-pillon{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.sm-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.sm-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sm-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sm-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sma" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="smah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sm-c" x="16" y="20">recvFD(c)</text><text class="sm-h" x="20" y="44">sender, its own table</text><rect class="sm-sys" x="20" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="110" y="52" width="30" height="28" rx="3"/><rect class="sm-hot" x="140" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="170" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="200" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="230" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="52" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="71" fill="#3a3a41">0</text><text class="sm-n" x="65" y="71" fill="#3a3a41">1</text><text class="sm-n" x="95" y="71" fill="#3a3a41">2</text><text class="sm-n" x="125" y="71" fill="#3a3a41">3</text><text class="sm-n" x="155" y="71" fill="#3a3a41">4</text><text class="sm-n" x="185" y="71" fill="#bab6ad">5</text><text class="sm-n" x="215" y="71" fill="#bab6ad">6</text><text class="sm-n" x="245" y="71" fill="#bab6ad">7</text><text class="sm-n" x="275" y="71" fill="#bab6ad">8</text><text class="sm-h" x="20" y="172">receiver, a table of its own</text><rect class="sm-sys" x="20" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="110" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="140" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="170" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="200" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="230" y="180" width="30" height="28" rx="3"/><rect class="sm-hot" x="260" y="180" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="199" fill="#3a3a41">0</text><text class="sm-n" x="65" y="199" fill="#3a3a41">1</text><text class="sm-n" x="95" y="199" fill="#3a3a41">2</text><text class="sm-n" x="125" y="199" fill="#bab6ad">3</text><text class="sm-n" x="155" y="199" fill="#3a3a41">4</text><text class="sm-n" x="185" y="199" fill="#3a3a41">5</text><text class="sm-n" x="215" y="199" fill="#3a3a41">6</text><text class="sm-n" x="245" y="199" fill="#3a3a41">7</text><text class="sm-n" x="275" y="199" fill="#3a3a41">8</text><rect class="sm-pillon" x="55" y="112" width="200" height="36" rx="18"/><text class="sm-pt" x="155" y="135">SCM_RIGHTS  [8]</text><rect class="sm-boxon" x="380" y="102" width="200" height="56" rx="4"/><text class="sm-lbl" x="480" y="124">open file description</text><text class="sm-n" x="480" y="146" fill="#3a3a41">offset 3</text><path class="sm-ln" d="M155,80 L376,120" marker-end="url(#sma)"/><path class="sm-hi" d="M275,180 L376,140" marker-end="url(#smah)"/><text class="sm-out" x="16" y="236">same message, and the number in it now reads 8</text></svg>
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 5. The receiver reads three bytes through slot 8 and gets DEF, carrying on from where the sender stopped. The offset moves to 6."><style>.sm-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sm-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sm-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sm-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sm-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sm-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sm-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sm-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-pill{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:4 3}.sm-pillon{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.sm-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.sm-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sm-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sm-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sma" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="smah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sm-c" x="16" y="20">receiver read(8, 3)</text><text class="sm-h" x="20" y="44">sender, its own table</text><rect class="sm-sys" x="20" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="110" y="52" width="30" height="28" rx="3"/><rect class="sm-hot" x="140" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="170" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="200" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="230" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="52" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="71" fill="#3a3a41">0</text><text class="sm-n" x="65" y="71" fill="#3a3a41">1</text><text class="sm-n" x="95" y="71" fill="#3a3a41">2</text><text class="sm-n" x="125" y="71" fill="#3a3a41">3</text><text class="sm-n" x="155" y="71" fill="#3a3a41">4</text><text class="sm-n" x="185" y="71" fill="#bab6ad">5</text><text class="sm-n" x="215" y="71" fill="#bab6ad">6</text><text class="sm-n" x="245" y="71" fill="#bab6ad">7</text><text class="sm-n" x="275" y="71" fill="#bab6ad">8</text><text class="sm-h" x="20" y="172">receiver, a table of its own</text><rect class="sm-sys" x="20" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="110" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="140" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="170" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="200" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="230" y="180" width="30" height="28" rx="3"/><rect class="sm-hot" x="260" y="180" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="199" fill="#3a3a41">0</text><text class="sm-n" x="65" y="199" fill="#3a3a41">1</text><text class="sm-n" x="95" y="199" fill="#3a3a41">2</text><text class="sm-n" x="125" y="199" fill="#bab6ad">3</text><text class="sm-n" x="155" y="199" fill="#3a3a41">4</text><text class="sm-n" x="185" y="199" fill="#3a3a41">5</text><text class="sm-n" x="215" y="199" fill="#3a3a41">6</text><text class="sm-n" x="245" y="199" fill="#3a3a41">7</text><text class="sm-n" x="275" y="199" fill="#3a3a41">8</text><rect class="sm-boxon" x="380" y="102" width="200" height="56" rx="4"/><text class="sm-lbl" x="480" y="124">open file description</text><text class="sm-n" x="480" y="146" fill="#3a3a41">offset 6</text><path class="sm-ln" d="M155,80 L376,120" marker-end="url(#sma)"/><path class="sm-hi" d="M275,180 L376,140" marker-end="url(#smah)"/><text class="sm-out" x="16" y="236">&quot;DEF&quot;   it carries on from the sender&#39;s position</text></svg>
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 6. The sender reads again and gets GHI. Two unrelated processes are taking turns on one position."><style>.sm-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.sm-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.sm-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.sm-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.sm-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.sm-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.sm-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.sm-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.sm-pill{fill:#fff;stroke:#c8c4bb;stroke-width:1.1;stroke-dasharray:4 3}.sm-pillon{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.sm-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.sm-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.sm-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.sm-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sma" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="smah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="sm-c" x="16" y="20">sender read(4, 3)</text><text class="sm-h" x="20" y="44">sender, its own table</text><rect class="sm-sys" x="20" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="52" width="30" height="28" rx="3"/><rect class="sm-sys" x="110" y="52" width="30" height="28" rx="3"/><rect class="sm-hot" x="140" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="170" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="200" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="230" y="52" width="30" height="28" rx="3"/><rect class="sm-off" x="260" y="52" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="71" fill="#3a3a41">0</text><text class="sm-n" x="65" y="71" fill="#3a3a41">1</text><text class="sm-n" x="95" y="71" fill="#3a3a41">2</text><text class="sm-n" x="125" y="71" fill="#3a3a41">3</text><text class="sm-n" x="155" y="71" fill="#3a3a41">4</text><text class="sm-n" x="185" y="71" fill="#bab6ad">5</text><text class="sm-n" x="215" y="71" fill="#bab6ad">6</text><text class="sm-n" x="245" y="71" fill="#bab6ad">7</text><text class="sm-n" x="275" y="71" fill="#bab6ad">8</text><text class="sm-h" x="20" y="172">receiver, a table of its own</text><rect class="sm-sys" x="20" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="50" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="80" y="180" width="30" height="28" rx="3"/><rect class="sm-off" x="110" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="140" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="170" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="200" y="180" width="30" height="28" rx="3"/><rect class="sm-sys" x="230" y="180" width="30" height="28" rx="3"/><rect class="sm-hot" x="260" y="180" width="30" height="28" rx="3"/><text class="sm-n" x="35" y="199" fill="#3a3a41">0</text><text class="sm-n" x="65" y="199" fill="#3a3a41">1</text><text class="sm-n" x="95" y="199" fill="#3a3a41">2</text><text class="sm-n" x="125" y="199" fill="#bab6ad">3</text><text class="sm-n" x="155" y="199" fill="#3a3a41">4</text><text class="sm-n" x="185" y="199" fill="#3a3a41">5</text><text class="sm-n" x="215" y="199" fill="#3a3a41">6</text><text class="sm-n" x="245" y="199" fill="#3a3a41">7</text><text class="sm-n" x="275" y="199" fill="#3a3a41">8</text><rect class="sm-boxon" x="380" y="102" width="200" height="56" rx="4"/><text class="sm-lbl" x="480" y="124">open file description</text><text class="sm-n" x="480" y="146" fill="#3a3a41">offset 9</text><path class="sm-hi" d="M155,80 L376,120" marker-end="url(#smah)"/><path class="sm-ln" d="M275,180 L376,140" marker-end="url(#sma)"/><text class="sm-out" x="16" y="236">&quot;GHI&quot;   two strangers, one position</text></svg>
</div>
<figcaption>Two tables that never met, one position, and a number rewritten in flight.</figcaption>
</figure>

The same result, between two processes with no ancestor in common. Now look at the numbers rather
than the letters, and trace the one message from both ends:

```
sender    sendmsg(...  SCM_RIGHTS, cmsg_data=[4])
receiver  recvmsg(...  SCM_RIGHTS, cmsg_data=[8])
```

One message, two numbers. The sender wrote 4 into it; by the time the receiver reads the same
control message it says 8, because the receiving kernel took the lowest free slot in the receiver's
own table and wrote that number in. Nothing was copied and nothing was translated by either program.
The descriptor sits in the out-of-band part of a message rather than the payload for exactly this
reason: it is not data, it is an instruction to install a reference.

</details>

**And one that looks like an exception but is not.** Linux does show the socket in `/proc`, which
seems to contradict the whole "no name" claim. The run earlier in this post already settled it: the
name is there, `socket:[53454226]`, and opening it comes back `ENXIO`. So the socket does have a
path, and you can read it. You just cannot open it. `/proc` gives you a name for looking, not a name
for reaching. Which is a fair summary of the whole arrangement:
the descriptor is the only handle that works, and it only works from inside the process that holds
it.

So the descriptor you hold is three levels away from the connection, and it is the only one of the
three that belongs to you.

## Where reuse turns into a bug

<p class="gloss">It happens when a number changes hands.</p>
Your event loop is handed an event that names `fd 5`. It looks up connection 5 and writes the
response. By then `fd 5` is a log file, and the reply for one user has gone somewhere no user will
ever read it. Every step was correct. The number was not.

This is the bug the three levels make possible, and it does not need bad luck to appear. A `dup`
makes it happen on every run, which turns it from something to argue about into something to watch.

<p class="gloss">epoll in one line: it hands back a tag you chose, not the file object.</p>
One term first, because the bug lives inside it. `epoll` is the kernel half of an event loop: you
register a set of connections, then ask it to block until some of them have something to read, and
it hands back the list of which ones. Anything serving many connections on few threads is sitting on
it, nginx and Node and the Go runtime included, usually far enough down that you never call it
yourself. The part that matters here is what "which ones" means. You do not get an object back. You
get a tag you chose at registration time, and almost everyone chooses the descriptor number.

So here is the reproduction. Wire the server end of a connection into `epoll`, `dup` it, close the
original number, open a file so the number gets handed out again, then send a byte down the socket
and see what the event says. `describeFd` reads `/proc` for what each number points at, so every
line reports what it found. Linux only.

```go
func main() {
  defer os.Remove("/tmp/fd-stale.log")

  epollFd := createEpollFd()
  serverFd, clientFd := openConnection()
  wireFdToEpoll(serverFd, epollFd)
  fmt.Printf("register serverFd        -> fd %d, %s\n", serverFd, describeFd(serverFd))

  keeperFd, _ := syscall.Dup(serverFd) // a second number for the same socket
  defer syscall.Close(keeperFd)
  syscall.Close(serverFd)
  fmt.Printf("dup, then close serverFd -> fd %d is %s\n", serverFd, describeFd(serverFd))

  logFd, _ := syscall.Open("/tmp/fd-stale.log", syscall.O_RDONLY|syscall.O_CREAT, 0o644)
  fmt.Printf("open a log file          -> fd %d, %s\n\n", logFd, describeFd(logFd))

  syscall.Write(clientFd, []byte("X"))
  fmt.Printf("client fd %d sends        -> to the socket, which is now only fd %d\n", clientFd, keeperFd)

  for _, fd := range waitForReadyFds(epollFd) {
    fmt.Printf("epoll_wait says          -> fd %d received an event\n", fd)
    fmt.Printf("the handler reads        -> fd %d, which is %s\n", fd, describeFd(fd))
  }
}
```

```
register serverFd        -> fd 5, socket:[54899389]
dup, then close serverFd -> fd 5 is nothing open at that number
open a log file          -> fd 5, /tmp/fd-stale.log

client fd 6 sends        -> to the socket, which is now only fd 7
epoll_wait says          -> fd 5 received an event
the handler reads        -> fd 5, which is /tmp/fd-stale.log
```

<figure class="steps">
<div class="steps-frames">
<svg viewBox="0 0 620 272" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 1. Slot 4 holds the epoll instance. Its watch list is drawn below the table and is empty."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ptl{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-rd{stroke:#a12027;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker><marker id="sear" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#a12027"/></marker></defs><text class="se-c" x="16" y="20">epollFd := createEpollFd()</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#bab6ad">5</text><text class="se-n" x="215" y="73" fill="#bab6ad">6</text><text class="se-n" x="245" y="73" fill="#bab6ad">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><path class="se-hi" d="M155,84 L155,190" marker-end="url(#seah)"/><text class="se-h" x="20" y="188">epoll&#39;s watch list</text><rect class="se-epoff" x="20" y="196" width="212" height="40" rx="4"/><text class="se-lbl" x="126" y="221">nothing registered yet</text><text class="se-out" x="16" y="262">epoll is reached through a number too, and its list starts empty</text></svg>
<svg viewBox="0 0 620 272" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 2. Slots 5 and 6 are filled by a socket pair: 5 is the server end, 6 is the client end."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ptl{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-rd{stroke:#a12027;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker><marker id="sear" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#a12027"/></marker></defs><text class="se-c" x="16" y="20">serverFd, clientFd := openConnection()</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#bab6ad">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-boxon" x="380" y="98" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="115">the client end</text><text class="se-n" x="490" y="132" fill="#3a3a41">the other end of the pair</text><rect class="se-boxon" x="380" y="162" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="179">the server end</text><text class="se-n" x="490" y="196" fill="#3a3a41">one struct file</text><path class="se-hi" d="M185,84 L376,178" marker-end="url(#seah)"/><path class="se-hi" d="M215,84 L376,118" marker-end="url(#seah)"/><path class="se-ln" d="M155,84 L155,190" marker-end="url(#sea)"/><text class="se-h" x="20" y="188">epoll&#39;s watch list</text><rect class="se-epoff" x="20" y="196" width="212" height="40" rx="4"/><text class="se-lbl" x="126" y="221">nothing registered yet</text><text class="se-out" x="16" y="262">fd 5 is the server end, fd 6 is the other end of the same connection</text></svg>
<svg viewBox="0 0 620 272" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 3. The watch list gains one entry. The descriptor is passed twice at registration: once to name the object to watch, and once as the tag to hand back. The entry stores the tag 5 and a pointer to the server socket."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ptl{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-rd{stroke:#a12027;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker><marker id="sear" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#a12027"/></marker></defs><text class="se-c" x="16" y="20">wireFdToEpoll(serverFd, epollFd)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#bab6ad">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-box" x="380" y="98" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="115">the client end</text><text class="se-n" x="490" y="132" fill="#3a3a41">the other end of the pair</text><rect class="se-boxon" x="380" y="162" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="179">the server end</text><text class="se-n" x="490" y="196" fill="#3a3a41">one struct file</text><path class="se-hi" d="M185,84 L376,178" marker-end="url(#seah)"/><path class="se-ln" d="M215,84 L376,118" marker-end="url(#sea)"/><path class="se-ln" d="M155,84 L155,190" marker-end="url(#sea)"/><text class="se-h" x="20" y="188">epoll&#39;s watch list</text><rect class="se-ep" x="20" y="196" width="212" height="40" rx="4"/><line x1="116" y1="197" x2="116" y2="235" stroke="#a12027" stroke-width="1"/><text class="se-pt" x="68" y="221">tag: 5</text><text class="se-lbl" x="174" y="221">the socket</text><path class="se-rd" d="M234,216 L376,198" marker-end="url(#sear)"/><text class="se-out" x="16" y="262">serverFd goes in twice: once to say what to watch, once as the tag to report</text></svg>
<svg viewBox="0 0 620 272" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 4. dup fills slot 7 with a second number pointing at the same server socket."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ptl{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-rd{stroke:#a12027;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker><marker id="sear" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#a12027"/></marker></defs><text class="se-c" x="16" y="20">keeperFd := dup(serverFd)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-box" x="380" y="98" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="115">the client end</text><text class="se-n" x="490" y="132" fill="#3a3a41">the other end of the pair</text><rect class="se-boxon" x="380" y="162" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="179">the server end</text><text class="se-n" x="490" y="196" fill="#3a3a41">one struct file</text><path class="se-ln" d="M185,84 L376,178" marker-end="url(#sea)"/><path class="se-ln" d="M215,84 L376,118" marker-end="url(#sea)"/><path class="se-hi" d="M245,84 L376,168" marker-end="url(#seah)"/><path class="se-ln" d="M155,84 L155,190" marker-end="url(#sea)"/><text class="se-h" x="20" y="188">epoll&#39;s watch list</text><rect class="se-ep" x="20" y="196" width="212" height="40" rx="4"/><line x1="116" y1="197" x2="116" y2="235" stroke="#a12027" stroke-width="1"/><text class="se-pt" x="68" y="221">tag: 5</text><text class="se-lbl" x="174" y="221">the socket</text><path class="se-rd" d="M234,216 L376,198" marker-end="url(#sear)"/><text class="se-out" x="16" y="262">fd 7 is a second number for the same socket</text></svg>
<svg viewBox="0 0 620 272" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 5. Slot 5 is empty. The socket survives because slot 7 still holds it, so the watch entry is unchanged."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ptl{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-rd{stroke:#a12027;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker><marker id="sear" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#a12027"/></marker></defs><text class="se-c" x="16" y="20">close(serverFd)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#bab6ad">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-box" x="380" y="98" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="115">the client end</text><text class="se-n" x="490" y="132" fill="#3a3a41">the other end of the pair</text><rect class="se-boxon" x="380" y="162" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="179">the server end</text><text class="se-n" x="490" y="196" fill="#3a3a41">still alive, held by fd 7</text><path class="se-ln" d="M215,84 L376,118" marker-end="url(#sea)"/><path class="se-ln" d="M245,84 L376,168" marker-end="url(#sea)"/><path class="se-ln" d="M155,84 L155,190" marker-end="url(#sea)"/><text class="se-h" x="20" y="188">epoll&#39;s watch list</text><rect class="se-ep" x="20" y="196" width="212" height="40" rx="4"/><line x1="116" y1="197" x2="116" y2="235" stroke="#a12027" stroke-width="1"/><text class="se-pt" x="68" y="221">tag: 5</text><text class="se-lbl" x="174" y="221">the socket</text><path class="se-rd" d="M234,216 L376,198" marker-end="url(#sear)"/><text class="se-out" x="16" y="262">slot 5 is free, the socket lives on fd 7, and the entry still says tag 5</text></svg>
<svg viewBox="0 0 620 272" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 6. Opening a file takes the lowest free number, which is 5 again. Slot 5 now points at the log file."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ptl{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-rd{stroke:#a12027;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker><marker id="sear" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#a12027"/></marker></defs><text class="se-c" x="16" y="20">open(&quot;/tmp/fd-stale.log&quot;)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-boxon" x="380" y="42" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="59">the log file</text><text class="se-n" x="490" y="76" fill="#3a3a41">never registered</text><rect class="se-box" x="380" y="98" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="115">the client end</text><text class="se-n" x="490" y="132" fill="#3a3a41">the other end of the pair</text><rect class="se-box" x="380" y="162" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="179">the server end</text><text class="se-n" x="490" y="196" fill="#3a3a41">still alive, held by fd 7</text><path class="se-hi" d="M185,52 L185,34 L470,34 L470,40" marker-end="url(#seah)"/><path class="se-ln" d="M215,84 L376,118" marker-end="url(#sea)"/><path class="se-ln" d="M245,84 L376,168" marker-end="url(#sea)"/><path class="se-ln" d="M155,84 L155,190" marker-end="url(#sea)"/><text class="se-h" x="20" y="188">epoll&#39;s watch list</text><rect class="se-ep" x="20" y="196" width="212" height="40" rx="4"/><line x1="116" y1="197" x2="116" y2="235" stroke="#a12027" stroke-width="1"/><text class="se-pt" x="68" y="221">tag: 5</text><text class="se-lbl" x="174" y="221">the socket</text><path class="se-rd" d="M234,216 L376,198" marker-end="url(#sear)"/><text class="se-out" x="16" y="262">the free number came back, and this time it is a file</text></svg>
<svg viewBox="0 0 620 272" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 7. The client end writes a byte. Only the server end receives it, so the watch entry fires, and it fires on the right object."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ptl{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-rd{stroke:#a12027;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker><marker id="sear" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#a12027"/></marker></defs><text class="se-c" x="16" y="20">syscall.Write(clientFd, []byte(&quot;X&quot;))</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-box" x="380" y="42" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="59">the log file</text><text class="se-n" x="490" y="76" fill="#3a3a41">never registered</text><rect class="se-boxon" x="380" y="98" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="115">the client end</text><text class="se-n" x="490" y="132" fill="#3a3a41">the other end of the pair</text><rect class="se-boxon" x="380" y="162" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="179">the server end</text><text class="se-n" x="490" y="196" fill="#3a3a41">a byte waiting to be read</text><path class="se-ln" d="M185,52 L185,34 L470,34 L470,40" marker-end="url(#sea)"/><path class="se-hi" d="M215,84 L376,118" marker-end="url(#seah)"/><path class="se-ln" d="M245,84 L376,168" marker-end="url(#sea)"/><path class="se-ln" d="M155,84 L155,190" marker-end="url(#sea)"/><text class="se-h" x="20" y="188">epoll&#39;s watch list</text><rect class="se-ep" x="20" y="196" width="212" height="40" rx="4"/><line x1="116" y1="197" x2="116" y2="235" stroke="#a12027" stroke-width="1"/><text class="se-pt" x="68" y="221">tag: 5</text><text class="se-lbl" x="174" y="221">the socket</text><path class="se-rd" d="M234,216 L376,198" marker-end="url(#sear)"/><path class="se-rd" d="M490,142 L490,156" marker-end="url(#sear)"/><text class="se-pt" x="506" y="155">X</text><text class="se-out" x="16" y="262">only the server end got the byte, so the entry fires, and it fires correctly</text></svg>
<svg viewBox="0 0 620 272" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 8. epoll_wait hands back the tag 5, and only the tag. Following that number now lands on the log file, while the byte is still on the socket, reachable only through fd 7."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ptl{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-rd{stroke:#a12027;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker><marker id="sear" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#a12027"/></marker></defs><text class="se-c" x="16" y="20">waitForReadyFds(epollFd)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-boxon" x="380" y="42" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="59">the log file</text><text class="se-n" x="490" y="76" fill="#3a3a41">what the handler reads</text><rect class="se-box" x="380" y="98" width="220" height="40" rx="4"/><text class="se-lbl" x="490" y="115">the client end</text><text class="se-n" x="490" y="132" fill="#3a3a41">the other end of the pair</text><rect class="se-box" x="380" y="162" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="179">the server end</text><text class="se-n" x="490" y="196" fill="#3a3a41">where the byte actually is</text><path class="se-rd" d="M185,52 L185,34 L470,34 L470,40" marker-end="url(#sear)"/><path class="se-ln" d="M215,84 L376,118" marker-end="url(#sea)"/><path class="se-ln" d="M245,84 L376,168" marker-end="url(#sea)"/><path class="se-ln" d="M155,84 L155,190" marker-end="url(#sea)"/><text class="se-h" x="20" y="188">epoll&#39;s watch list</text><rect class="se-ep" x="20" y="196" width="212" height="40" rx="4"/><line x1="116" y1="197" x2="116" y2="235" stroke="#a12027" stroke-width="1"/><text class="se-pt" x="68" y="221">tag: 5</text><text class="se-lbl" x="174" y="221">the socket</text><path class="se-rd" d="M226,194 L186,86" marker-end="url(#sear)"/><text class="se-ptl" x="238" y="165">returns tag 5</text><text class="se-out" x="16" y="262">right about the object, wrong about the name: tag 5 is the log file now</text></svg>
</div>
<figcaption>Red is epoll only: its watch list, its entry, and the number it hands back.</figcaption>
</figure>

<details class="tryit" markdown="1">
<summary>The five helpers, and the raw calls inside them</summary>

```go
const (
  waitTimeoutMs    = 1000
  maxEventsPerWait = 4 // a ceiling on one call, not a quota it waits to fill
)

// the epoll instance is itself reached through a descriptor
func createEpollFd() int {
  epollFd, _ := syscall.EpollCreate1(0)
  return epollFd
}

func openConnection() (serverFd, clientFd int) {
  pair, _ := syscall.Socketpair(syscall.AF_UNIX, syscall.SOCK_STREAM, 0)
  return pair[0], pair[1]
}

func wireFdToEpoll(fd, epollFd int) {
  syscall.EpollCtl(epollFd, syscall.EPOLL_CTL_ADD, fd, &syscall.EpollEvent{
    Events: syscall.EPOLLIN, // report it when it has something to read
    Fd:     int32(fd),       // the tag handed back when it fires
  })
}

func waitForReadyFds(epollFd int) []int {
  events := make([]syscall.EpollEvent, maxEventsPerWait)
  n, _ := syscall.EpollWait(epollFd, events, waitTimeoutMs)
  var tags []int
  for i := 0; i < n; i++ {
    tags = append(tags, int(events[i].Fd))
  }
  return tags
}

func describeFd(fd int) string {
  name, err := os.Readlink(fmt.Sprintf("/proc/self/fd/%d", fd))
  if err != nil {
    return "nothing open at that number"
  }
  return name
}
```

</details>

The last three lines do not agree, and the disagreement is the bug. Nothing in the program is
exotic: registering a connection, closing it, opening something else, and reading the ready list are
the four things an event loop does all day.

Three ordinary rules line up to produce it, and none of them is a mistake on its own.

**The number comes back at once.** POSIX requires the lowest available one
([§2.6](https://pubs.opengroup.org/onlinepubs/9799919799/functions/V2_chap02.html)), and on Linux the
number you just released is the first one the next `open()` will try.

**The entry outlives the number.** Closing a descriptor does not take the registration out. It goes
when the last reference to the socket goes, and `keeperFd` is still holding one. So the entry
registered through `fd 5` was still in the list when the byte arrived, and it is the one that fired.

**The number in the event is one you handed over.** The descriptor goes into the registration twice,
and only one of the two is a descriptor to the kernel:

```go
syscall.EpollCtl(epollFd, syscall.EPOLL_CTL_ADD, fd, &syscall.EpollEvent{
  Events: syscall.EPOLLIN, // report it when it has something to read
  Fd:     int32(fd),       // the tag handed back when it fires
})
```

The third argument is what to watch, and the kernel resolves it to that socket right there. `Fd`
inside the struct is `epoll_data`, storage the kernel keeps and returns untouched, and almost
everyone fills it with the descriptor. So the event is right about the object and wrong about the
name.

Those three explain the wrong reply, and the same field is the fix. Store something that cannot be
recycled behind your back: an id you allocate for the connection and never hand out twice, or the
descriptor paired with a counter you bump on every close, checked before you act. Then a stale event
fails a comparison instead of finding a stranger.

<details class="tryit" markdown="1">
<summary>Extra: what if the watcher is a different process?</summary>

Nothing above says the two references have to be in one program. Send the server end to an unrelated
process over a Unix socket, let both sides watch it with their own `epoll`, and write one byte. Two
questions come apart here that usually travel together: who gets told, and who gets the bytes.

One binary, started twice, exactly like `passfd` above, and `sendFD`, `recvFD` and `dial` are the
same three helpers from that run. The handshakes matter: whoever reads first destroys the evidence
for everyone else, so each side has to report what it was told before anyone is allowed to read.

```go
func main() {
  if len(os.Args) > 1 && os.Args[1] == "send" {
    send()
    return
  }
  receive()
}

// its own epoll instance, watching the descriptor this process holds
func watch(fd int) int {
  ep, _ := syscall.EpollCreate1(0)
  syscall.EpollCtl(ep, syscall.EPOLL_CTL_ADD, fd, &syscall.EpollEvent{
    Events: syscall.EPOLLIN,
    Fd:     int32(fd),
  })
  return ep
}

func send() {
  pair, _ := syscall.Socketpair(syscall.AF_UNIX, syscall.SOCK_STREAM, 0)
  serverFd, clientFd := pair[0], pair[1]
  ep := watch(serverFd)

  c := dial(sock)
  sendFD(c, serverFd) // the receiver gets its own number for this same socket
  c.Read(make([]byte, 1))

  fmt.Printf("one byte written to the other end\n")
  syscall.Write(clientFd, []byte("X"))
  fmt.Printf("sender   -> epoll told it: %v\n", told(ep))

  c.Write([]byte{0})      // your turn to report
  c.Read(make([]byte, 1)) // both have looked, nobody has read yet
  fmt.Printf("sender   -> read: %s\n", tryRead(serverFd))
  c.Write([]byte{0})
  c.Read(make([]byte, 1)) // wait for the receiver's line before exiting
}

func receive() {
  os.Remove(sock)
  l, _ := net.ListenUnix("unix", &net.UnixAddr{Name: sock, Net: "unix"})
  defer os.Remove(sock)

  c, _ := l.AcceptUnix()
  fd := recvFD(c)
  ep := watch(fd)
  c.Write([]byte{0}) // registered, safe to write the byte now

  saw := told(ep)         // look before anyone reads
  c.Read(make([]byte, 1)) // print only when the sender has had its turn
  fmt.Printf("receiver -> epoll told it: %v\n", saw)
  c.Write([]byte{0}) // looked, still have not read

  c.Read(make([]byte, 1)) // the sender has taken its read
  fmt.Printf("receiver -> read: %s\n", tryRead(fd))
  c.Write([]byte{0})
}
```

```
one byte written to the other end
sender   -> epoll told it: true
receiver -> epoll told it: true
sender   -> read: "X"
receiver -> read: nothing (resource temporarily unavailable)
```

Both are told. One gets the byte. Readiness is a property of the socket, so every interest list
watching it fires, in as many processes as hold a descriptor for it. The bytes are not copied per
watcher: they sit in the one receive queue, so reading is a race and the loser gets `EAGAIN`. That
is the thundering herd, and it is what `EPOLLEXCLUSIVE` was added to control.

Waiting on something is not the same as owning what arrives on it.

</details>

## What makes it worse

<p class="gloss">The surprise is here: <code>close()</code> is not a cancel.</p>
The misdirected reply is the symptom you can see. Two more things follow from that surviving entry,
and the run below produces both.

Register `fd 5` with `epoll`, then register `fd 5` a second time. The second call fails: it is
already in there. Which makes epoll look like it is keeping track of the connection, and asking
twice is simply a mistake on your side. `fd 7` is that same connection under a different number, so
it should fail the same way. The tag `4242` marks the first registration, so the rows can be told
apart later.

```go
func main() {
  epollFd, _ := syscall.EpollCreate1(0)
  pair, _ := syscall.Socketpair(syscall.AF_UNIX, syscall.SOCK_STREAM, 0)
  serverFd, clientFd := pair[0], pair[1]
  keeperFd, _ := syscall.Dup(serverFd)
  drain := make([]byte, 1)

  fmt.Printf("fd %d and fd %d name one socket\n\n", serverFd, keeperFd)
  fmt.Printf("register fd %d as tag 4242  -> %s\n", serverFd,
    outcome(wireFdToEpollAsTag(serverFd, epollFd, 4242), "accepted"))
  fmt.Printf("register fd %d once more    -> %s\n", serverFd,
    outcome(wireFdToEpollAsTag(serverFd, epollFd, 9999), "accepted"))
  fmt.Printf("register fd %d as tag %d     -> %s\n", keeperFd, keeperFd,
    outcome(wireFdToEpollAsTag(keeperFd, epollFd, int32(keeperFd)), "accepted"))
  fmt.Printf("the interest list          -> %s\n", interestList(epollFd))
  syscall.Write(clientFd, []byte("X"))
  fmt.Printf("one write                  -> tags %v\n\n", waitForReadyTags(epollFd))
  syscall.Read(keeperFd, drain)

  syscall.Close(serverFd)
  fmt.Printf("close fd %d                 -> DEL fd %d: %s\n", serverFd, serverFd,
    outcome(unwireFdFromEpoll(serverFd, epollFd), "removed"))

  var fresh [2]int
  syscall.Pipe(fresh[:])
  fmt.Printf("a pipe retakes fd %d        -> DEL fd %d: %s\n", fresh[0], fresh[0],
    outcome(unwireFdFromEpoll(fresh[0], epollFd), "removed"))

  fmt.Printf("the interest list          -> %s\n", interestList(epollFd))
  syscall.Write(clientFd, []byte("Y"))
  fmt.Printf("one write                  -> tags %v\n\n", waitForReadyTags(epollFd))
  syscall.Read(keeperFd, drain)

  fmt.Printf("DEL fd %d                   -> %s\n", keeperFd,
    outcome(unwireFdFromEpoll(keeperFd, epollFd), "removed"))
  fmt.Printf("the interest list          -> %s\n", interestList(epollFd))
  syscall.Write(clientFd, []byte("Z"))
  fmt.Printf("one write                  -> tags %v\n\n", waitForReadyTags(epollFd))
  syscall.Read(keeperFd, drain)

  syscall.Close(keeperFd)
  fmt.Printf("close fd %d                 -> the socket's last reference\n", keeperFd)
  fmt.Printf("the interest list          -> %s\n", interestList(epollFd))
}
```

```
fd 5 and fd 7 name one socket

register fd 5 as tag 4242  -> accepted
register fd 5 once more    -> file exists
register fd 7 as tag 7     -> accepted
the interest list          -> (fd 5, tag 4242) (fd 7, tag 7)
one write                  -> tags [7 4242]

close fd 5                 -> DEL fd 5: bad file descriptor
a pipe retakes fd 5        -> DEL fd 5: no such file or directory
the interest list          -> (fd 5, tag 4242) (fd 7, tag 7)
one write                  -> tags [7 4242]

DEL fd 7                   -> removed
the interest list          -> (fd 5, tag 4242)
one write                  -> tags [4242]

close fd 7                 -> the socket's last reference
the interest list          -> empty
```

<figure class="steps">
<div class="steps-frames">
<svg viewBox="0 0 620 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 1. Slots 5 and 7 both point at one socket. The interest list below the table is empty."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="se-c" x="16" y="20">keeperFd, _ := syscall.Dup(serverFd)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-boxon" x="380" y="100" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="117">the socket</text><text class="se-n" x="490" y="134" fill="#3a3a41">one struct file</text><path class="se-hi" d="M185,84 L376,136" marker-end="url(#seah)"/><path class="se-hi" d="M245,84 L376,120" marker-end="url(#seah)"/><text class="se-h" x="20" y="124">epoll&#39;s interest list</text><rect class="se-epoff" x="20" y="132" width="300" height="44" rx="4"/><text class="se-lbl" x="170" y="159">empty</text><text class="se-out" x="16" y="252">two numbers, one socket, and nothing watching it yet</text></svg>
<svg viewBox="0 0 620 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 2. One row appears, keyed on the socket paired with fd 5, and reporting the tag 4242. Adding the same descriptor a second time is refused as a duplicate."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="se-c" x="16" y="20">wireFdToEpollAsTag(serverFd, epollFd, 4242)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-boxon" x="380" y="100" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="117">the socket</text><text class="se-n" x="490" y="134" fill="#3a3a41">one struct file</text><path class="se-hi" d="M185,84 L376,136" marker-end="url(#seah)"/><path class="se-ln" d="M245,84 L376,120" marker-end="url(#sea)"/><text class="se-h" x="20" y="124">epoll&#39;s interest list</text><rect class="se-ep" x="20" y="132" width="300" height="44" rx="4"/><line x1="200" y1="133" x2="200" y2="175" stroke="#a12027" stroke-width="1"/><line x1="21" y1="152" x2="319" y2="152" stroke="#a12027" stroke-width="1"/><text class="se-lbl" x="110" y="146">key</text><text class="se-lbl" x="260" y="146">reports</text><text class="se-pt" x="110" y="169">socket, fd 5</text><text class="se-pt" x="260" y="169">4242</text><text class="se-out" x="16" y="252">one row: keyed on the socket and fd 5, reporting a number no fd has</text><text class="se-out" x="16" y="268">the same fd a second time -> file exists</text></svg>
<svg viewBox="0 0 620 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 3. Registering the same socket under fd 7 is accepted rather than rejected, so the list now holds two rows for one socket."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="se-c" x="16" y="20">wireFdToEpollAsTag(keeperFd, epollFd, 7)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-boxon" x="380" y="100" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="117">the socket</text><text class="se-n" x="490" y="134" fill="#3a3a41">one struct file</text><path class="se-ln" d="M185,84 L376,136" marker-end="url(#sea)"/><path class="se-hi" d="M245,84 L376,120" marker-end="url(#seah)"/><text class="se-h" x="20" y="124">epoll&#39;s interest list</text><rect class="se-ep" x="20" y="132" width="300" height="68" rx="4"/><line x1="200" y1="133" x2="200" y2="199" stroke="#a12027" stroke-width="1"/><line x1="21" y1="152" x2="319" y2="152" stroke="#a12027" stroke-width="1"/><text class="se-lbl" x="110" y="146">key</text><text class="se-lbl" x="260" y="146">reports</text><text class="se-pt" x="110" y="169">socket, fd 5</text><text class="se-pt" x="260" y="169">4242</text><line x1="21" y1="176" x2="319" y2="176" stroke="#a12027" stroke-width="1" stroke-opacity="0.4"/><text class="se-pt" x="110" y="193">socket, fd 7</text><text class="se-pt" x="260" y="193">7</text><text class="se-out" x="16" y="252">accepted, not a duplicate: the key is the pair, so both rows stand</text></svg>
<svg viewBox="0 0 620 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 4. One write makes the socket readable. Both rows match it, so a single byte is reported twice, as tags 7 and 4242."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="se-c" x="16" y="20">syscall.Write(clientFd, []byte(&quot;X&quot;))</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-boxon" x="380" y="100" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="117">the socket</text><text class="se-n" x="490" y="134" fill="#3a3a41">a byte waiting to be read</text><path class="se-ln" d="M185,84 L376,136" marker-end="url(#sea)"/><path class="se-ln" d="M245,84 L376,120" marker-end="url(#sea)"/><text class="se-h" x="20" y="124">epoll&#39;s interest list</text><rect class="se-ep" x="20" y="132" width="300" height="68" rx="4"/><line x1="200" y1="133" x2="200" y2="199" stroke="#a12027" stroke-width="1"/><line x1="21" y1="152" x2="319" y2="152" stroke="#a12027" stroke-width="1"/><text class="se-lbl" x="110" y="146">key</text><text class="se-lbl" x="260" y="146">reports</text><text class="se-pt" x="110" y="169">socket, fd 5</text><text class="se-pt" x="260" y="169">4242</text><line x1="21" y1="176" x2="319" y2="176" stroke="#a12027" stroke-width="1" stroke-opacity="0.4"/><text class="se-pt" x="110" y="193">socket, fd 7</text><text class="se-pt" x="260" y="193">7</text><text class="se-out" x="16" y="252">one write, both rows match, so one byte comes back as two events</text><text class="se-out" x="16" y="268">epoll_wait -> tags [7 4242]</text></svg>
<svg viewBox="0 0 620 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 5. Slot 5 is empty but both rows survive, and the number can no longer be used to remove its own row."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="se-c" x="16" y="20">syscall.Close(serverFd)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#bab6ad">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#bab6ad">8</text><rect class="se-boxon" x="380" y="100" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="117">the socket</text><text class="se-n" x="490" y="134" fill="#3a3a41">still alive, held by fd 7</text><path class="se-ln" d="M245,84 L376,120" marker-end="url(#sea)"/><text class="se-h" x="20" y="124">epoll&#39;s interest list</text><rect class="se-ep" x="20" y="132" width="300" height="68" rx="4"/><line x1="200" y1="133" x2="200" y2="199" stroke="#a12027" stroke-width="1"/><line x1="21" y1="152" x2="319" y2="152" stroke="#a12027" stroke-width="1"/><text class="se-lbl" x="110" y="146">key</text><text class="se-lbl" x="260" y="146">reports</text><text class="se-pt" x="110" y="169">socket, fd 5</text><text class="se-pt" x="260" y="169">4242</text><line x1="21" y1="176" x2="319" y2="176" stroke="#a12027" stroke-width="1" stroke-opacity="0.4"/><text class="se-pt" x="110" y="193">socket, fd 7</text><text class="se-pt" x="260" y="193">7</text><text class="se-out" x="16" y="252">slot 5 is free and both rows still stand</text><text class="se-out" x="16" y="268">DEL fd 5 -> bad file descriptor</text></svg>
<svg viewBox="0 0 620 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 6. A pipe takes the free numbers 5 and 8. Removing by the number 5 now looks for a row keyed on the pipe, and finds none."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="se-c" x="16" y="20">syscall.Pipe(fresh[:])</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#3a3a41">8</text><rect class="se-boxon" x="380" y="42" width="220" height="38" rx="4"/><text class="se-lbl" x="490" y="59">a new pipe</text><text class="se-n" x="490" y="76" fill="#3a3a41">took fd 5 and fd 8</text><rect class="se-box" x="380" y="100" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="117">the socket</text><text class="se-n" x="490" y="134" fill="#3a3a41">still alive, held by fd 7</text><path class="se-hi" d="M185,52 L185,34 L470,34 L470,40" marker-end="url(#seah)"/><path class="se-ln" d="M245,84 L376,120" marker-end="url(#sea)"/><text class="se-h" x="20" y="124">epoll&#39;s interest list</text><rect class="se-ep" x="20" y="132" width="300" height="68" rx="4"/><line x1="200" y1="133" x2="200" y2="199" stroke="#a12027" stroke-width="1"/><line x1="21" y1="152" x2="319" y2="152" stroke="#a12027" stroke-width="1"/><text class="se-lbl" x="110" y="146">key</text><text class="se-lbl" x="260" y="146">reports</text><text class="se-pt" x="110" y="169">socket, fd 5</text><text class="se-pt" x="260" y="169">4242</text><line x1="21" y1="176" x2="319" y2="176" stroke="#a12027" stroke-width="1" stroke-opacity="0.4"/><text class="se-pt" x="110" y="193">socket, fd 7</text><text class="se-pt" x="260" y="193">7</text><text class="se-out" x="16" y="252">the free number came back, and it is a pipe</text><text class="se-out" x="16" y="268">DEL fd 5 -> no such file or directory: that key would be (pipe, 5)</text></svg>
<svg viewBox="0 0 620 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 7. Removing by fd 7 takes out only that row. The row keyed on fd 5 remains and still fires."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="se-c" x="16" y="20">unwireFdFromEpoll(keeperFd, epollFd)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-hot" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#3a3a41">7</text><text class="se-n" x="275" y="73" fill="#3a3a41">8</text><rect class="se-box" x="380" y="42" width="220" height="38" rx="4"/><text class="se-lbl" x="490" y="59">a new pipe</text><text class="se-n" x="490" y="76" fill="#3a3a41">took fd 5 and fd 8</text><rect class="se-boxon" x="380" y="100" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="117">the socket</text><text class="se-n" x="490" y="134" fill="#3a3a41">still alive, held by fd 7</text><path class="se-ln" d="M185,52 L185,34 L470,34 L470,40" marker-end="url(#sea)"/><path class="se-hi" d="M245,84 L376,120" marker-end="url(#seah)"/><text class="se-h" x="20" y="124">epoll&#39;s interest list</text><rect class="se-ep" x="20" y="132" width="300" height="44" rx="4"/><line x1="200" y1="133" x2="200" y2="175" stroke="#a12027" stroke-width="1"/><line x1="21" y1="152" x2="319" y2="152" stroke="#a12027" stroke-width="1"/><text class="se-lbl" x="110" y="146">key</text><text class="se-lbl" x="260" y="146">reports</text><text class="se-pt" x="110" y="169">socket, fd 5</text><text class="se-pt" x="260" y="169">4242</text><text class="se-out" x="16" y="252">removed, and the row left behind has no name that reaches it</text><text class="se-out" x="16" y="268">one write -> tags [4242]</text></svg>
<svg viewBox="0 0 620 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 8. Closing fd 7 destroys the socket, and the remaining row disappears with it, leaving the interest list empty."><style>.se-c{font:12px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#3a3a41}.se-h{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82}.se-lbl{font:11px ui-sans-serif,system-ui,sans-serif;fill:#7a7a82;text-anchor:middle}.se-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;text-anchor:middle}.se-sys{fill:#f4f2ee;stroke:#d8d4cb;stroke-width:1.1}.se-off{fill:#fff;stroke:#d8d4cb;stroke-width:1;stroke-dasharray:3 3}.se-hot{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-box{fill:#fff;stroke:#c8c4bb;stroke-width:1.1}.se-boxon{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.se-ep{fill:#fdf3f4;stroke:#a12027;stroke-width:1.4}.se-epoff{fill:#fff;stroke:#c8c4bb;stroke-width:1;stroke-dasharray:4 3}.se-pt{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#a12027;text-anchor:middle}.se-ln{stroke:#c8c4bb;stroke-width:1.1;fill:none}.se-hi{stroke:#2f7d75;stroke-width:1.4;fill:none}.se-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><defs><marker id="sea" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#c8c4bb"/></marker><marker id="seah" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L7,3.5 L0,7 z" fill="#2f7d75"/></marker></defs><text class="se-c" x="16" y="20">syscall.Close(keeperFd)</text><text class="se-h" x="20" y="46">your descriptor table</text><rect class="se-sys" x="20" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="50" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="80" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="110" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="140" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="170" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="200" y="54" width="30" height="28" rx="3"/><rect class="se-off" x="230" y="54" width="30" height="28" rx="3"/><rect class="se-sys" x="260" y="54" width="30" height="28" rx="3"/><text class="se-n" x="35" y="73" fill="#3a3a41">0</text><text class="se-n" x="65" y="73" fill="#3a3a41">1</text><text class="se-n" x="95" y="73" fill="#3a3a41">2</text><text class="se-n" x="125" y="73" fill="#3a3a41">3</text><text class="se-n" x="155" y="73" fill="#3a3a41">4</text><text class="se-n" x="185" y="73" fill="#3a3a41">5</text><text class="se-n" x="215" y="73" fill="#3a3a41">6</text><text class="se-n" x="245" y="73" fill="#bab6ad">7</text><text class="se-n" x="275" y="73" fill="#3a3a41">8</text><rect class="se-box" x="380" y="42" width="220" height="38" rx="4"/><text class="se-lbl" x="490" y="59">a new pipe</text><text class="se-n" x="490" y="76" fill="#3a3a41">took fd 5 and fd 8</text><rect class="se-off" x="380" y="100" width="220" height="44" rx="4"/><text class="se-lbl" x="490" y="117">the socket</text><text class="se-n" x="490" y="134" fill="#3a3a41">gone with its last reference</text><path class="se-ln" d="M185,52 L185,34 L470,34 L470,40" marker-end="url(#sea)"/><text class="se-h" x="20" y="124">epoll&#39;s interest list</text><rect class="se-epoff" x="20" y="132" width="300" height="44" rx="4"/><text class="se-lbl" x="170" y="159">empty</text><text class="se-out" x="16" y="252">the last reference goes, and the row goes with it</text></svg>
</div>
<figcaption>The interest list as procfs reports it: one row per entry, key and reported number.</figcaption>
</figure>

<details class="tryit" markdown="1">
<summary>How the list is read, and the other helpers</summary>

```go
func wireFdToEpollAsTag(fd, epollFd int, tag int32) error {
  return syscall.EpollCtl(epollFd, syscall.EPOLL_CTL_ADD, fd, &syscall.EpollEvent{
    Events: syscall.EPOLLIN,
    Fd:     tag,
  })
}

func unwireFdFromEpoll(fd, epollFd int) error {
  return syscall.EpollCtl(epollFd, syscall.EPOLL_CTL_DEL, fd, nil)
}

func waitForReadyTags(epollFd int) []int32 {
  events := make([]syscall.EpollEvent, 8)
  n, _ := syscall.EpollWait(epollFd, events, 300)
  tags := []int32{}
  for i := 0; i < n; i++ {
    tags = append(tags, events[i].Fd)
  }
  return tags
}

// procfs prints one line per interest-list entry, so the list can be read
// rather than inferred from what fires.
func interestList(epollFd int) string {
  raw, _ := os.ReadFile(fmt.Sprintf("/proc/self/fdinfo/%d", epollFd))
  var rows []string
  for _, line := range strings.Split(string(raw), "\n") {
    if !strings.HasPrefix(line, "tfd:") {
      continue
    }
    f := strings.Fields(line)
    tag, _ := strconv.ParseUint(f[5], 16, 64) // the kernel prints data in hex
    rows = append(rows, fmt.Sprintf("(fd %s, tag %d)", f[1], tag))
  }
  if rows == nil {
    return "empty"
  }
  return strings.Join(rows, " ")
}

func outcome(err error, ok string) string {
  if err != nil {
    return err.Error()
  }
  return ok
}
```

</details>

Two of the three answers are what turn a misdirected reply into something worse.

**One write arrives as two events.** The same descriptor twice is a duplicate and refused, the same
socket under a second descriptor is not, so what the kernel matches on is the pair. One socket
carries two rows, and one byte coming in is reported once per row.

**Registering a descriptor is a promise to keep it.** The row keyed on `fd 5` keeps firing once
`fd 5` is closed, and that number can no longer reach it: removing by it fails while it is free, and
finds no such row once a new file has taken it. Removing by `fd 7` takes out only `fd 7`'s row. The
key is still the pair, so there is one way back: put the same socket at the number again, with a
`dup2` from a reference you still hold, and the removal matches and succeeds. It works even if
something else has taken the number, because `dup2` closes whatever is there first. What it needs is
the second handle, which is exactly what you do not have if your close was the last one.

The third answer is the one behind the original bug: the tag `4242` comes back exactly as it went
in, so the kernel never looked a descriptor up on the way out. And the list itself is read from
`/proc/self/fdinfo` rather than guessed at from what fires.

So cancel the registration before you close the descriptor, never after. Afterwards you are relying
on a second handle you may not have, and on whether `close()` happened to be the last reference,
which depends on something you may not control: whether anyone else is still holding the same file.

<details class="tryit" markdown="1">
<summary>Where the kernel writes the pair down</summary>

Everything above is one struct. This is the key an interest-list row is filed under, and the
comments are the comparison order:

```c
/* struct epoll_filefd in fs/eventpoll.c */
struct epoll_filefd {
  struct file *file;   // compared first
  int fd;              // only breaks ties
} __packed;
```

[`struct epoll_filefd`](https://github.com/torvalds/linux/blob/v7.1/fs/eventpoll.c#L102) ·
[`ep_cmp_ffd`](https://github.com/torvalds/linux/blob/v7.1/fs/eventpoll.c#L351)

Two rows for one socket follow from the second field: same file, different number, so the rows do
not collide. `fd 5` staying in a key after `fd 5` is closed follows from the first: the row is filed
under the socket, and the socket is still there.

Which leaves what does remove a row. Not `close()` as such: removal hangs off
[`eventpoll_release`](https://github.com/torvalds/linux/blob/v7.1/include/linux/eventpoll.h#L38),
which `__fput()` calls when the last reference to the file drops, so `close()` clears a row only on
the occasion that it happens to be that last reference. The comment above that function says why it
exists at all, which is to clean up "files that are closed without being removed from the eventpoll
interface". The case this section is about is the case the kernel wrote the function for.

</details>

**kqueue disagrees about all of this.** I assumed it was the same idea with different spelling. On
XNU, fd-based registrations hang off the descriptor table and are indexed by the number itself:

```c
/* knote_fdfind in bsd/kern/kern_event.c */
if (is_fd) {
  /* fd-based knotes are linked off the fd table */
  if (kev->kei_ident < (u_int)fdp->fd_knlistsize) {
    list = &fdp->fd_knlist[kev->kei_ident];
  }
} else if (fdp->fd_knhashmask != 0) {
  /* hash non-fd knotes here too */
  list = &fdp->fd_knhash[KN_HASH((u_long)kev->kei_ident, fdp->fd_knhashmask)];
}
```

[`knote_fdfind`](https://github.com/apple-oss-distributions/xnu/blob/xnu-11215.41.3/bsd/kern/kern_event.c#L6782)

Matching is on the descriptor number plus the filter. Two operating systems, two different answers
to "what is being watched". That is why portable code over both has to choose semantics, not just
syscalls.

## So: is everything a file?

Yes, and more literally than the slogan suggests. It is not that the kernel treats things *like*
files. It builds a real `struct file` for them and hangs a table of functions off it saying what the
verbs mean. A socket is a file because someone wrote `socket_file_ops`.

Which is also why the abstraction has a visible edge. Ask epoll to watch a file on disk and it
refuses:

```c
/* do_epoll_ctl in fs/eventpoll.c */
if (!file_can_poll(fd_file(tf)))
  return -EPERM;
```

[`do_epoll_ctl`](https://github.com/torvalds/linux/blob/v7.1/fs/eventpoll.c#L2257-L2258)

Regular files do not implement poll, so `epoll_ctl` rejects them with `EPERM`. Sockets and pipes
have a readiness question worth asking. A file on disk is always "ready", and the wait you actually
care about is the one the disk imposes, which readiness has no way to express.

So the honest version of the slogan is narrower and more useful. Everything is a file in the sense
that everything gets the same *handle*: a number, an entry in your table, and an object underneath
with an ops table. Everything is not a file in the sense of behaving alike, and the sharp edges of
this whole subject are exactly where that difference surfaces. A read position that two handles
share. A number that comes back the moment you free it. An epoll registration that outlives the
descriptor you closed. None of those are quirks. They are what it costs to put one interface over
things that have nothing else in common.

## What this buys you

A short list of what the runs above actually settled.

- **The number is an index, and nothing more.** It is a slot in a per-process table, so the same
  number means different things in two processes, and `/proc/self/fd` is where you look it up.
- **`dup` copies at a different level than `open`.** Two descriptors from `dup` share one read
  position; two `open()` calls on one path do not. That is the whole reason the middle box exists.
- **Sharing crosses processes.** `fork` and a descriptor passed over a Unix socket both land on the
  same open file description, so the shared position is not a parent-and-child special case.
- **Sockets really are files.** The kernel builds a `struct file` and hangs an ops table off it.
  `/proc` will even name the socket, and that name cannot be opened.
- **The number you close is the next one handed out.** Not eventually, immediately, and POSIX
  requires it.
- **`epoll` reports a tag you chose, not an object.** The entry is keyed on the file *and* the
  descriptor, it survives closing that descriptor, and once the number is gone you cannot cancel it
  by that number alone.
- **Readiness is per-file, delivery is per-queue.** Every watcher of a socket is told it is ready;
  only one of them gets the bytes.
- **`2>&1 > file` and `> file 2>&1` differ for one reason.** Redirections apply left to right and
  each copies whatever the slot holds at that moment, so a copy is not a link. Worth deriving once
  instead of memorising.

The last one generalises past shells. Anything handing out small reusable integers inherits it:
connection ids, session ids, slots in an object pool. The fix has one shape, a counter beside the
number that moves when the slot is recycled, so a stale reference can be recognised rather than
prevented. Go's runtime keeps exactly that counter beside every descriptor it watches
([`pollDesc.fdseq`](https://github.com/golang/go/blob/015343854b5d9e2829481df30dbcae2ca6682d25/src/runtime/netpoll.go#L79)).

## Sources

- **POSIX.1-2024**: [§2.6 File Descriptor
  Allocation](https://pubs.opengroup.org/onlinepubs/9799919799/functions/V2_chap02.html) for the
  lowest-available rule, and
  [`open()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/open.html) for the
  definition of "open file description".
- **Linux man pages**: [`socket(2)`](https://man7.org/linux/man-pages/man2/socket.2.html),
  [`accept(2)`](https://man7.org/linux/man-pages/man2/accept.2.html),
  [`epoll(7)`](https://man7.org/linux/man-pages/man7/epoll.7.html).
- **Books**: Kerrisk, *The Linux Programming Interface*. Stevens, *UNIX Network Programming,
  Volume 1*.

### The source, pinned

Every line referenced above, at the revision it was read: Linux `v7.1`, XNU `xnu-11215.41.3`, Go
`01534385`, busybox `1_36_1`.

- `fs/file.c`: [`alloc_fd`](https://github.com/torvalds/linux/blob/v7.1/fs/file.c#L570),
  [`find_next_fd`](https://github.com/torvalds/linux/blob/v7.1/fs/file.c#L544),
  [`__put_unused_fd`](https://github.com/torvalds/linux/blob/v7.1/fs/file.c#L626)
- `fs/eventpoll.c`: [`struct epoll_filefd`](https://github.com/torvalds/linux/blob/v7.1/fs/eventpoll.c#L102),
  [`ep_cmp_ffd`](https://github.com/torvalds/linux/blob/v7.1/fs/eventpoll.c#L351),
  [`eventpoll_release_file`](https://github.com/torvalds/linux/blob/v7.1/fs/eventpoll.c#L1110)
- `bsd/kern/kern_event.c`: [`knote_fdfind`](https://github.com/apple-oss-distributions/xnu/blob/xnu-11215.41.3/bsd/kern/kern_event.c#L6782)
  (a source release, not the kernel Apple ships, so the line numbers are pinned to that release)
