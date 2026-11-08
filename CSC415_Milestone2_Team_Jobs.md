# Milestone 2: Team Jobs and Instructions ("Directories")

**Team:** Jake, Mike, Quynh, Trieu
**Builds on:** our Milestone 1 repo (the version we submitted on Nov 4)
**Turned in?** No. But Milestone 3 needs all of this, and the final writeup has an **"Approach Milestone 2"** section.
**Lecture links:** Ch.13 *Tree-Structured Directories* (absolute and relative path names, the current directory, operations on a directory: search, create, delete, list) and Ch.14 *Directory Implementation* (a linear list of entries). The slides have no step-by-step list for this milestone like Milestone 1 had, so the steps below are ours.
**With code:** the same plan with all the tested code is in `CSC415_Milestone2_Team_Plan.md`.

---

## 1. What must work

**Shell commands** (we turn them ON in `fsshell.c`):
- ☐ `md` (make directory)
- ☐ `ls`, `ls -a`, `ls -l` (list directory)
- ☐ `cd` (change directory)
- ☐ `pwd` (print working directory)
- ☐ `rm` (optional, but we do it)
- ☐ After the shell restarts, all directories are still there.

**Functions** (the prototypes are already in the professor's `mfs.h`):

| Function | What it does | Who |
|---|---|---|
| `fs_setcwd` | `cd`: change the current directory | Jake |
| `fs_getcwd` | `pwd`: give the current directory as text | Jake |
| `fs_opendir`, `fs_readdir`, `fs_closedir` | `ls`: go through a directory one entry at a time | Mike |
| `fs_stat` | `ls -l`: size and dates of one entry | Mike |
| `fs_isDir`, `fs_isFile` | "is this a directory / a file?" | Mike |
| `fs_mkdir` | `md`: make a new directory | Trieu |
| `fs_rmdir`, `fs_delete` (optional) | `rm`: remove an empty directory / a file | Quynh |

**Writeup:** the template's "Approach Milestone 2" asks for our plan for *directory functions, path parsing, directory iteration, current working directory behavior, and file/directory identification*. Each person writes one short paragraph about their part (section 11).

---

## 2. Key words

| Term | Simple meaning |
|---|---|
| **Path** | The "address" of a file or directory, like `/docs/notes`. The parts are split by `/`. |
| **Absolute path** | Starts with `/`, so we start at the **root**. Example: `/docs/notes`. |
| **Relative path** | Does not start with `/`, so we start at the **current directory**. Example: `notes`, `../pics`. |
| **cwd** | Current working directory: where you "are" now. `cd` changes it. `pwd` prints it. |
| **"." and ".."** | Real entries in every directory. "." = this directory. ".." = the parent. |
| **parsePath** | Our function that turns a path into "which directory has it, and where is it inside". Everyone uses it. |
| **Parent** | The directory that holds the last name. In `/docs/notes`, the parent is `docs`. |
| **Last name** | The last part of the path. In `/docs/notes`, it is `notes`. |
| **Index** | Which entry (0, 1, 2 ...) the last name is in the parent. **-1 = not there.** |
| **Load a directory** | Read its block from the disk into memory with `LBAread`, so we can look at the entries. |
| **fdDir** | A small struct (like a **bookmark**) that remembers where `ls` is while it reads a directory. |
| **Contract** | The prototypes in a `.h` file. Others can write their code against them before your `.c` file is done. |

---

## 3. The big idea

### 3.1 How we find a path

A directory is a **list** of entries. To find a name, we check the entries one by one (the lecture's *linear list*).

After `md docs`, `md pics` and `md docs/notes`, the disk looks like this:

| root (block 154) | docs (block 155) | notes (block 157) |
|---|---|---|
| 0: "." → 154 | 0: "." → 155 | 0: "." → 157 |
| 1: ".." → 154 | 1: ".." → 154 | 1: ".." → 155 |
| 2: docs → 155 (directory) | 2: notes → 157 (directory) | 2 – 7: free |
| 3: pics → 156 (directory) | 3 – 7: free | |
| 4 – 7: free | | |

Finding `/docs/notes`:

| Step | Looking in | Looking for | Found? | What happens |
|---|---|---|---|---|
| 1 | root (154) | docs | yes, index 2, it is a directory | `notes` comes after it, so go down: load block 155 |
| 2 | docs (155) | notes | yes, index 2 | `notes` is the **last** part, so stop |
| Result | | | | parent = docs (in memory), last name = "notes", index = 2 |

- If the **last** name is not there, parsePath still says "ok", with **index = -1**. That is what `md` wants: "the parent is fine, and the name is free".
- If a **middle** part is missing or is not a directory (like `/nothere/x`), parsePath says **bad path (-1)**.

### 3.2 Our 5 design rules

1. **The cwd is a block number + a text.** We do **not** keep a copy of the cwd directory in memory. We keep *where* it is (its start block) and its text for `pwd`. parsePath loads it fresh from the disk every time, so it is **never out of date** (for example, after `md` changes it).
2. **parsePath gives you a parent, and you must free it.** This is the one memory rule for everyone. Every function that calls parsePath frees the parent at the end.
3. **The path `/` means "the root's own `.` entry".** parsePath gives parent = root, index = 0, last name = ".". Then `cd /`, `ls /` and `fs_isDir("/")` work with no special code, and `md /` says "already exists".
4. **One directory = 1 block = 8 entries** (".", ".." and 6 free), the same as the root. So reading or writing a directory is one `LBAread` or one `LBAwrite`. (Limit: 6 items per directory. See section 12.)
5. **After every change, save at once.** `md` and `rm` change the FAT (they allocate or release blocks). They call `saveVolume` right away to write the FAT and the VCB. If someone closes the shell with Ctrl-C, the volume is still correct.

---

## 4. Who does what

| Member | Files (only this person edits them) | Functions | Test with | Writeup paragraph | Reviewer |
|---|---|---|---|---|---|
| **Trieu** | `directory.h`, `directory.c` (add), `mkdir.c` (new), `tests/testDirectory.c` | 6 helpers + `fs_mkdir` | `tests/testDirectory.c`, then `md` in the shell | Directory functions | Jake |
| **Jake** | `path.h`, `path.c` (new), `tests/testPath.c` | `parsePath`, `fs_setcwd`, `fs_getcwd`, `initPaths`, `getCwdStart` | `tests/testPath.c` | Path parsing + cwd | Mike |
| **Mike** | `dirRead.c` (new), `mfs.h` (only the `fdDir` struct), `tests/testDirRead.c` | `fs_opendir`, `fs_readdir`, `fs_closedir`, `fs_stat`, `fs_isDir`, `fs_isFile` | `tests/testDirRead.c` | Directory iteration + file/dir identification | Quynh |
| **Quynh** | `fsInit.h` (new), `fsInit.c`, `remove.c` (new), `Makefile`, `fsshell.c` (only the `_ON` lines) | `saveVolume`, `fs_rmdir`, `fs_delete`, connecting everything | `make run` | Main flow + rm | Trieu |

Each reviewer **uses** the code they review, so they read it carefully.

**Who needs whom** (keep the waiting short):

| Order | Part | Who needs it |
|---|---|---|
| 1st (Day 2) | Trieu's helpers in `directory.c` | everyone |
| 2nd (Day 2 – 3) | Jake's `parsePath` in `path.c` | Mike, Trieu's `md`, Quynh's `rm` |
| 3rd (Day 3) | Mike's `dirRead.c`, Trieu's `mkdir.c`, Quynh's `remove.c` | the shell |
| 4th (Day 4) | Quynh: `initPaths` call + Makefile + shell switches | the full test |

- **Day 1, everyone commits their `.h` change first** (the contracts). After that, anyone can compile their own `.c` file alone, even if the other `.c` files are not done.
- Trieu's helpers are short. **Trieu pushes them on Day 2**, so Jake and Mike can run their tests.
- Jake's and Mike's tests build a small tree **by hand** on their own `TestVolume`. They do **not** need the free space code or Quynh's code.
- `md` and `rm` need everything, so they are tested in the shell after Quynh connects it all (Day 4).

---

## 5. Rules for everyone

1. Edit **only your own files**. Then there are no merge conflicts.
2. `git pull` before you start. Add only your own files by name (never `git add .`).
3. **After you change a `.h` file, run `make clean`, then `make`.** Our Makefile does not see changes in `.h` files.
4. Your code must build before every commit. **Until Day 4, `make` does not build the new `.c` files** (Quynh adds them to the Makefile on Day 4). So also compile your own file alone with warnings on (`gcc -c -Wall -I.` + your file) before you commit.
5. Make one commit for each function (or small group of functions). Make **3 or more commits each**.
6. Commit only `.c`, `.h` and `Makefile`. Do **not** commit `fsshell`, `.o` files, test programs, `SampleVolume` or `TestVolume`. After testing, run `git restore SampleVolume`.
7. Each test makes its own `TestVolume`. Delete it after testing.
8. To build a test alone: your test file + the `.c` files it needs + `fsLow.o` (on Apple Silicon, `fsLowM1.o`) + the math library (`-lm`). The list for each test is in your section.
9. Every new file starts with **your** class header on line 1: `CSC-415-03 Fall 2026`, your name, your student ID, your GitHub name, `Combo Number 4`, `Basic File System`, the file name, and 1 – 3 lines about the file.
10. Simple, clear code: short functions, clear names, short comments that explain **why**.

---

## 6. Trieu: directory helpers + `md`

**The idea:** a directory is like a **page in a phone book**. To find a name, read the lines one by one. To add a name, use the first empty line. `md` = make a new page, then write its name on a free line of the parent's page.

**Files:** `directory.h`, `directory.c` (add to the end), `mkdir.c` (new), `tests/testDirectory.c` (add to it)

### Step 1 (Day 1): the contract in `directory.h`
Add 6 prototypes before `#endif`. Also fix the header line in `directory.h` and `directory.c` to `Combo Number 4`.

| Function | Gets | Gives back |
|---|---|---|
| `loadDirectory` | a start block, the block size | the directory in memory (the caller frees it), or NULL |
| `directoryEntryCount` | a directory | how many entries it has |
| `isUsedEntry` | one entry | 1 if used, 0 if free |
| `isDirectory` | one entry | 1 if it is a used directory, 0 if not |
| `findEntryInDir` | a directory, a name | the index of that name, or -1 |
| `findFreeEntry` | a directory | the first free index, or -1 if full |

### Step 2 (Day 2): the helpers in `directory.c`

| Function | How it works |
|---|---|
| `loadDirectory` | Refuse a start block below 1 (block 0 is the VCB, -1 means "no block"). Get one block of memory and read the first block. Check that entry 0 is named "." (if not, it is not a directory, so give back NULL). Use the size in "." to see if it has more blocks; if yes, read all of them (next to each other, the same rule as `writeDirectory`). |
| `directoryEntryCount` | The size in "." divided by 64. For our directories: 512 ÷ 64 = **8**. |
| `isUsedEntry` | A free entry has an **empty name**. Anything else is used. |
| `isDirectory` | Used, **and** bit 9 (the directory flag, 0x200) of `permission` is on. |
| `findEntryInDir` | Go through every entry. Skip the free ones. Compare the name. Give back the first match, or -1. |
| `findFreeEntry` | Start at entry **2** (0 and 1 are always "." and ".."). Give back the first free one, or -1. |

### Step 3 (Day 2): test it alone (`tests/testDirectory.c`)
Keep the Milestone 1 part. Add a new part: make a folder "a" **by hand** at block 160 inside the root (put a copy of a's "." into the root's first free entry, name it "a", write the root). Then read the root back with `loadDirectory` and check the helpers. Also make a 20-entry directory at block 170 to check that `loadDirectory` reads more than one block.

**Build with:** `tests/testDirectory.c` + `directory.c` + `fsLow.o`

| Check | You should see |
|---|---|
| Milestone 1 part | 1 block; entry 0 "." start 154, size 512; entry 1 ".." start 154, size 512; entry 2 "" start -1 |
| `directoryEntryCount` of the root | 8 |
| `findEntryInDir` "a" / "zzz" | 2 / -1 |
| `findFreeEntry` of the root | 3 |
| `isDirectory` on "a" | 1 |
| a's ".." | points to block 154 |
| a 20-entry directory | loads with 24 entries (3 blocks) |
| `loadDirectory(0)` | NULL (it prints "bad start block 0") |

**Push the helpers now.** Jake and Mike need them for their tests.

### Step 4 (Day 3): `fs_mkdir` in `mkdir.c`
Include `mfs.h`, `vcb.h`, `freeSpace.h`, `directory.h`, `path.h` and `fsInit.h`. Do the steps in this order:

| Step | What to do | If it fails |
|---|---|---|
| 1 | `parsePath` the path. | print "md: bad path …", give back -1 |
| 2 | The index must be **-1** (the name is new). | print "md: … already exists", free the parent, give back -1 |
| 3 | `findFreeEntry` in the parent. | print "md: the directory is full", free the parent, give back -1 |
| 4 | Ask `directoryBlocksNeeded` (8 entries, = 1 block), then `allocBlocks` that many. | print "md: no free space", free the parent, give back -1 |
| 5 | `createDirectory` with **the parent** and the new start block. (It builds "." and ".." and writes the new directory.) | `releaseBlocks` (give the block back), free the parent, give back -1 |
| 6 | In the parent's free entry, put a **copy of the new directory's "."**, then write the real name over it. Set the parent's "." modified time. `writeDirectory` the parent. | |
| 7 | The FAT changed, so call `saveVolume`. Free the new directory and the parent. Give back the result. | |

**Why step 6 copies the new ".":** it already has the right start block, size, dates and permission. The parent's entry needs the same facts, just with the real name. (Our test uses the same trick.)
The `mode` from the shell is ignored: every directory gets 0755, like the root.

### Done when
- ☐ Your test gives the results above.
- ☐ On Day 4, `md docs`, `md docs/notes` and `md pics` work, and `md pics` again says "already exists" (section 10).

**Trieu's commits (example):** `Add Milestone 2 prototypes to directory.h` → `Add loadDirectory and directoryEntryCount` → `Add isUsedEntry, isDirectory, findEntryInDir, findFreeEntry` → `Test the Milestone 2 helpers in testDirectory.c` → `Add fs_mkdir in mkdir.c`

---

## 7. Jake: `parsePath`, `cd` and `pwd`

**The idea:** a path is like **directions in a building**: "go to floor docs, then room notes". A path that starts with `/` starts at the front door (the root). Anything else starts where you are standing now (the cwd). We follow the directions one step at a time.

**Files:** `path.h`, `path.c` (both new), `tests/testPath.c`

### Step 1 (Day 1): the contract in `path.h`
- Include `directory.h`.
- A constant **MAX_PATH = 4096** (the same as the shell's `DIRMAX_LEN`).
- A struct **PathInfo**, which parsePath fills in:

| Field | Meaning |
|---|---|
| `parent` | the directory (in memory) that has, or will get, the last name |
| `index` | where the last name is in `parent`; **-1 = not there** |
| `lastName` | the last part of the path (36 characters, like a directory entry name) |

- 3 prototypes: `initPaths` (cwd = the root, "/"), `parsePath` (0 = ok, -1 = bad path), `getCwdStart` (the start block of the cwd; Quynh's `rm` uses it).

### Step 2 (Day 2): `initPaths`, `getCwdStart`, `parsePath` in `path.c`
Keep 2 private variables: the cwd's **start block** and the cwd's **text** (starts as "/").
- `initPaths`: cwd start block = the root's start block from the VCB, text = "/".
- `getCwdStart`: give back the cwd start block.

**`parsePath`, in this order:**

| Step | What to do |
|---|---|
| 1 | Refuse an empty path or a path of 4096 characters or more (-1). |
| 2 | Make a copy of the path, because `strtok_r` changes the string. |
| 3 | Pick the start: path starts with `/` → the root's start block (from the VCB); else → the cwd start block. **Load** that directory. |
| 4 | Cut off the first name with `strtok_r` (split on `/`). If there is no name, the path was just `/`: give back parent = the root, index = **0**, last name = ".". |
| 5 | Loop: if the name is 36 characters or more, free and give back -1. Find the name in the parent (`findEntryInDir`). Cut off the **next** name. |
| 6 | No next name → this name is the **last** one: fill in parent, index (can be -1), last name. Give back 0. |
| 7 | There is a next name → this one is in the middle, so it must exist **and** be a directory. If not: free the parent, give back -1. If yes: load it, free the old parent, it is the new parent. Go back to step 5 with the next name. |

"." and ".." need **no special code**: they are real entries on the disk, so `findEntryInDir` finds them.

### Step 3 (Day 3): `cd` and `pwd`
**`makeCleanPath`** (private helper, for the `pwd` text): put the cwd text in front if the path is relative. Then go through the parts **like a stack**: a name is pushed, ".." pops one (at the root it stays at the root), "." is skipped. Glue the parts back with `/`. No parts left = "/". Too long = -1.
Example: cwd `/a/b` + path `../c/./` → `/a/c`.

**`fs_setcwd`:**
1. `parsePath` the path. It must be found (index not -1) and be a directory. If not, give back -1.
2. Remember its start block, then free the parent.
3. Build the clean text with `makeCleanPath`.
4. **Only now** change the cwd start block and text. (So if anything fails, the cwd stays the same.) Give back 0.

**`fs_getcwd`:** if the caller's buffer is too small (or missing), give back NULL. Else copy the cwd text into it and give back the buffer.

### Step 4 (Day 3): test it alone (`tests/testPath.c`)
Make a VCB **in memory only** (load it, init it with 19,531 blocks, set the root start to 154). Build the tree **by hand**: root at 154, "a" at 155 inside the root, "b" at 156 inside "a". Then call `initPaths` and try the paths and `cd`.

**Build with:** `tests/testPath.c` + `path.c` + `directory.c` + `vcb.c` + `fsLow.o`

| parsePath of | You should see |
|---|---|
| `/` | ok, parent block 154, last ".", index 0 |
| `a` | ok, parent block 154, last "a", index 2 |
| `/a/b` | ok, parent block 155, last "b", index 2 |
| `a/b/..` | ok, parent block 156, last "..", index 1 |
| `/a/new` | ok, parent block 155, last "new", index **-1** |
| `/x/y` | bad path |
| `/a/b/c/d` | bad path |

| cd | Result | pwd after |
|---|---|---|
| `a/b` | 0 | /a/b |
| `..` | 0 | /a |
| `/a/./b/../..` | 0 | / |
| `nothere` | -1 | / (no change) |
| `/a/b` | 0 | /a/b |
| `/` | 0 | / |

### Done when
- ☐ Your test gives the results above.
- ☐ On Day 4, `cd`, `cd ../..` and `pwd` work in the shell (section 10).

**Jake's commits (example):** `Add path.h` → `Add initPaths, getCwdStart and parsePath` → `Add makeCleanPath, fs_setcwd and fs_getcwd` → `Add tests/testPath.c`

---

## 8. Mike: reading directories

**The idea:** `ls` reads a directory **like a book with a bookmark**. `fs_opendir` opens the book and puts the bookmark at page 0. Each `fs_readdir` gives the next **used** entry and moves the bookmark. `fs_closedir` closes the book (frees everything).

| Call | Gives back | Bookmark after |
|---|---|---|
| `fs_opendir("/docs")` | the bookmark (`fdDir`) | 0 |
| `fs_readdir` | "." | 1 |
| `fs_readdir` | ".." | 2 |
| `fs_readdir` | "notes" | 3 |
| `fs_readdir` | NULL (entries 3 – 7 are free, so we skip them; the end) | 8 |
| `fs_closedir` | 0 | |

**Files:** `dirRead.c` (new), `mfs.h` (only the `fdDir` struct), `tests/testDirRead.c`

### Step 1 (Day 1): the contract in `mfs.h`
The professor left a "TO DO" in the `fdDir` struct. Make **2 small changes** and nothing else:
1. Under the line that includes `b_io.h`, also include `directory.h`.
2. Replace the commented-out `DE * directory` line with a real field: a **pointer to our `DirectoryEntry`**, named `directory`. (It holds the loaded directory.)

The other fields stay: `d_reclen`, `dirEntryPosition` (the bookmark), and `di` (the item we give back).

### Step 2 (Day 2): `fs_opendir`, `fs_readdir`, `fs_closedir` in `dirRead.c`
Include `mfs.h`, `vcb.h`, `directory.h` and `path.h`.

| Function | How it works |
|---|---|
| `fs_opendir` | `parsePath`. It must be found and be a directory; else free and give back NULL. Remember its start block, free the parent, **load** the directory. Get memory for one `fdDir` and one `fs_diriteminfo`. (If anything fails, free what you got and give back NULL.) Set the bookmark to 0, keep the directory and the item in the `fdDir`. |
| `fs_readdir` | NULL in → NULL out. While the bookmark is before the end: take that entry, **move the bookmark**, and if the entry is used: copy its name into the item, set the type (`FT_DIRECTORY` or `FT_REGFILE`), set the record length, give back the item. At the end: NULL. |
| `fs_closedir` | Free the directory, the item and the `fdDir`. Give back 0 (-1 for NULL). |

### Step 3 (Day 3): `fs_stat`, `fs_isDir`, `fs_isFile`

| Function | How it works |
|---|---|
| `fs_stat` | `parsePath`; it must be found. Fill in: size = the entry's size; block size = from the VCB; blocks = size in **512-byte** blocks, rounded up (so 1000 bytes = 2); access time = modified time (we have no "accessed" field); modified time; create time. Free the parent. Give back 0 (or -1). |
| `fs_isDir` | `parsePath`. 1 if found **and** a directory, else 0. A bad path is 0. Free the parent. |
| `fs_isFile` | The same, but found **and not** a directory. |

These two give back **0** for a path that is not there. The shell's `rm` and `ls` use that to decide what to do.

### Step 4 (Day 3): test it alone (`tests/testDirRead.c`)
Same setup as Jake's test (VCB in memory, root 154, "a" 155, "b" 156 by hand). Also add a **fake file** to the root: a copy of the root's ".", named "notes.txt", size 1000, start block 200, permission 0644 (no directory flag). It needs Trieu's helpers and Jake's `parsePath`, so run it after they push. (Until then, just compile `dirRead.c` alone.)

**Build with:** `tests/testDirRead.c` + `dirRead.c` + `path.c` + `directory.c` + `vcb.c` + `fsLow.o`

| Check | You should see |
|---|---|
| list `/` | . (directory), .. (directory), a (directory), notes.txt (file) |
| list `/a` | . , .. , b (all directories) |
| `fs_isDir("a")` / `fs_isDir("notes.txt")` | 1 / 0 |
| `fs_isFile("notes.txt")` / `fs_isFile("nothere")` | 1 / 0 |
| stat notes.txt | size 1000, blocks 2, block size 512 |
| stat /a/b | size 512, blocks 1, block size 512 |
| `fs_opendir("notes.txt")` | NULL |

### Done when
- ☐ Your test gives the results above.
- ☐ On Day 4, `ls`, `ls -a` and `ls -l` work in the shell (section 10).

**Mike's commits (example):** `Fill in fdDir in mfs.h` → `Add fs_opendir, fs_readdir and fs_closedir` → `Add fs_stat, fs_isDir and fs_isFile` → `Add tests/testDirRead.c`

---

## 9. Quynh: connecting everything + `rm`

**The idea:** Quynh's code is the **glue**, like in Milestone 1. She adds one save function that everyone can call, the `rm` command, and on Day 4 she turns everything on.

**Files:** `fsInit.h` (new), `fsInit.c`, `remove.c` (new), `Makefile`, `fsshell.c` (only the `_ON` lines)

### Step 1 (Day 1): the contract in `fsInit.h`
One prototype: **`saveVolume`**. It writes the FAT and the VCB to disk. 0 = ok, -1 = error.

### Step 2 (Day 2): `fsInit.c`, includes + `saveVolume`
1. Also include `path.h` and `fsInit.h`.
2. Add **`saveVolume`** right after `copyFreeSpaceToVCB`. In this order: `copyFreeSpaceToVCB` → `writeFreeSpace` (if it fails: print "Error: could not save the free space map.", give back -1) → `writeVCB` (if it fails: print "Error: could not save the VCB.", give back -1) → give back 0. It is the same save code `exitFileSystem` already had, now in one function that `md` and `rm` can call too.
3. In `exitFileSystem`, replace the old save lines with one call to `saveVolume`. Then `cleanupFreeSpace`, `cleanupVCB`, "System exiting" as before.

`make` and `make run` still work the same after this step.

### Step 3 (Day 3): `fs_rmdir` and `fs_delete` in `remove.c`
Include `mfs.h`, `vcb.h`, `freeSpace.h`, `directory.h`, `path.h` and `fsInit.h`. Two private helpers:

| Helper | How it works |
|---|---|
| `isDirectoryEmpty` | Entries 2 to the end must all be free. (Only "." and ".." may be used.) |
| `removeEntry` | If the entry has a start block (not -1), give the blocks back with `releaseBlocks`. (An empty file has no blocks.) Clear the entry to all zeros, then set its start block to -1 (a free entry, the same as `createDirectory` makes). Set the parent's "." modified time. `writeDirectory` the parent. Then `saveVolume`, because the FAT changed. |

**`fs_rmdir`, check in this order:**

| Check | If it fails, print |
|---|---|
| `parsePath` works | "rm: bad path …" |
| found **and** a directory | "rm: … is not a directory" |
| the last name is **not** "." or ".." (this also blocks `rm /`) | "rm: cannot remove …" |
| it is **not the cwd** (compare with `getCwdStart`) | "rm: cannot remove the current directory" |
| load it: it is **empty** | "rm: … is not empty" |

All ok → `removeEntry`. Always free the parent (and the loaded directory) before you give back.

**`fs_delete`** (for files; files come in Milestone 3, but `rm` is ready): `parsePath`; it must be found and **not** a directory ("rm: … is not a file"); then `removeEntry`.

### Step 4 (Day 4): connect it all (after everyone has pushed)
Do these together, because `initPaths` is in Jake's file, and the build needs every new file:
1. **fsInit.c:** at the end of `initFileSystem`, just before the last "give back 0", call `initPaths`. (So we always start in the root, "/".)
2. **Makefile:** add `path.o`, `dirRead.o`, `mkdir.o` and `remove.o` to the end of the `ADDOBJ` line.
3. **fsshell.c:** turn these 5 from 0 to 1: `CMDLS_ON`, `CMDMD_ON`, `CMDRM_ON`, `CMDCD_ON`, `CMDPWD_ON`. Leave the others at 0 (they need Milestone 3).
4. `make clean`, `make`, then the full test (section 10).

### Done when
- ☐ `make` has no warnings, and the command table at start shows ls, cd, md, pwd and rm **ON**.
- ☐ Run 1 and run 2 in section 10 give the results shown.

**Quynh's commits (example):** `Add fsInit.h with saveVolume` → `Add saveVolume to fsInit.c` → `Add fs_rmdir and fs_delete in remove.c` → `Add the new files to the Makefile and call initPaths` → `Turn on ls, md, rm, cd and pwd in fsshell`

---

## 10. Test everything together (Day 4)

`git pull` → `make clean` → `make` (no warnings) → delete `SampleVolume` → `make run`.

### Run 1: a new volume

| You type | You should see |
|---|---|
| (start) | "The volume is not formatted. Formatting now." and the command table with ls, cd, md, pwd, rm **ON** |
| `md docs`, `md pics`, `md docs/notes` | nothing (no error) |
| `ls` | docs, pics |
| `ls -l` | D 512 docs / D 512 pics |
| `cd docs`, then `pwd` | /docs |
| `ls -a` | . / .. / notes |
| `cd notes`, then `pwd` | /docs/notes |
| `cd ../..`, then `pwd` | / |
| `cd nothere` | Could not change path to nothere |
| `md pics` | md: pics already exists |
| `exit` | System exiting |

### Run 2: is it still there? (and `rm`)
`make run` again (do **not** delete `SampleVolume`).

| You type | You should see |
|---|---|
| (start) | "The volume is already formatted. Loading it." |
| `pwd` | / |
| `ls` | docs, pics |
| `ls docs` | notes |
| `rm docs` | rm: docs is not empty |
| `rm docs/notes` | nothing (it worked) |
| `rm pics` | nothing (it worked) |
| `ls` | docs |
| `exit` | System exiting |

Also try: `rm .`, `rm ..` and `rm /` (all say "cannot remove"), and `cd docs` then `rm /docs` ("cannot remove the current directory").

### Check with hexdump (optional, good for the writeup)
After `md docs`, `md pics` and `md docs/notes` on a new volume:

| Hexdump block | What you should find |
|---|---|
| 155 (the root) | entry 2 "docs" with start block `9B` (155), entry 3 "pics" with `9C` (156) |
| 156 (docs) | "." → `9B` (155), ".." → `9A` (154, the root), entry 2 "notes" → `9D` (157) |
| 1 (the VCB) | freeBlocks `AD 4B` = 19,373 (3 fewer than 19,376), freeListHead `9E` = 158 |

### Memory check (optional)
`make vrun`, do a few `md`, `ls`, `cd` and `rm`, then `exit`. Look for **"definitely lost: 0 bytes"**. (Valgrind shows 1 error inside the professor's `fsLow` when it makes a **new** volume. That one is not ours.)

---

## 11. The writeup: "Approach Milestone 2"

The template asks for the **plan before coding**: *directory functions, path parsing, directory iteration, current working directory behavior, and file/directory identification*. Each person writes one short paragraph (like Milestone 1):

| Who | Write about |
|---|---|
| **Trieu** | Directory functions: the linear list, the 6 helpers, how `md` adds a new directory to its parent, why 1 block |
| **Jake** | Path parsing and the cwd: absolute vs. relative, walking down one name at a time, why the cwd is a block number + text |
| **Mike** | Directory iteration and file/dir identification: the bookmark, skipping free entries, the directory flag |
| **Quynh** | The main flow and `rm`: `initPaths` at startup, `saveVolume` after every change, the rules for `rm` |

Also add each new function to the **Function Descriptions** table, and any real problem to **Issues and Resolutions**.

---

## 12. Known limits (for Milestone 3)

| Limit | Why | Later fix |
|---|---|---|
| A directory holds only 6 items (8 entries) | One directory = 1 block (rule 4) | Make directories grow: add a block, and read/write **block by block** following the FAT with `nextBlock` |
| `ls -l docs` (another folder) shows the wrong type and a random size | The professor's shell calls `fs_stat` and `fs_isDir` with only the name, **without** the folder, so they look in the cwd | `cd docs` first, then `ls -l`. Plain `ls docs` is fine. |
| `md` ignores the mode (0777) | Every directory gets 0755, like the root | Use the mode if we need it |
| No files yet | `touch`, `cat`, `cp` need `b_open`, `b_read`, `b_write` | Milestone 3. `fs_delete` is already ready for files. |

---

## 13. Schedule

These are suggested dates. Milestone 2 is not turned in, but Milestone 3 needs it, so we finish it in one week.

| Day | Trieu | Jake | Mike | Quynh |
|---|---|---|---|---|
| **1 (Sun Nov 8)** | `directory.h` prototypes | `path.h` | `mfs.h` fdDir | `fsInit.h` |
| **2 (Mon Nov 9)** | helpers + test, **push by evening** | `parsePath` | opendir / readdir / closedir | `saveVolume` in fsInit.c |
| **3 (Tue Nov 10)** | `fs_mkdir` | `cd` / `pwd` + testPath | stat / isDir / isFile + testDirRead | `remove.c` |
| **4 (Wed Nov 11)** | test `md` in the shell | test `cd` / `pwd` in the shell | test `ls` in the shell | `initPaths` + Makefile + shell switches (morning), then the full test |
| **5 (Thu Nov 12)** | fix bugs together at our meeting: run 1, run 2, `make vrun`; reviewers read the code | | | |
| **6 (Fri Nov 13)** | each person writes their writeup paragraph (section 11) | | | |

---

## 14. Final checklist

- ☐ `make clean` then `make`: no warnings
- ☐ The command table at start shows ls, cd, md, pwd and rm **ON**
- ☐ `md`, `ls`, `ls -a`, `ls -l`, `cd`, `cd ../..` and `pwd` work like run 1, and `cd /` goes back to the root
- ☐ After a restart, the directories are still there (run 2)
- ☐ `rm` refuses a non-empty directory, ".", "..", "/" and the cwd
- ☐ The three tests in `tests/` give the results in sections 6, 7 and 8
- ☐ Each person has 3 or more commits on their own files
- ☐ `git restore SampleVolume` before every commit, and no `TestVolume` committed
