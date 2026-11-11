# Milestone 3: Team Jobs and Instructions ("Files" + Final Submission)

**Team:** Jake, Mike, Quynh, Trieu

**Builds on:** our Milestone 2 code (directories: `md`, `ls`, `cd`, `pwd`, `rm` working in the shell)

**Turned in?** **Yes. This is the final group submission.** Due **Thu Nov 26, 11:59pm**, **1 attempt only**. We plan to submit early, on **Fri Nov 20** (latest Sun Nov 22), before the Thanksgiving break.

**From the professor's README:** we must write `b_open`, `b_read`, `b_write`, `b_seek`, `b_close`. `b_open` must support `O_CREAT`, `O_TRUNC`, `O_APPEND`, `O_RDONLY`, `O_WRONLY`, `O_RDWR`. The shell's `touch`, `cat`, `cp`, `cp2l`, `cp2fs` then work by themselves. `mv` is a **TODO for us** in `fsshell.c`.

---

## 0. Before we start: finish Milestone 2

Milestone 3 needs the professor's `ParsePath`, Jake's path helpers (`freeParent`, `getTargetEntry`, `refreshLoadedDirs`), Trieu's directory helpers (`LoadDirectory`, `isDEaDir`, `findFreeEntry` …) and the full Milestone 2 test. Milestone 3 uses **the same names and rules** as the updated Milestone 2 plan (section 3.2 there).

- ☐ Everyone pushed their Milestone 2 code (`path.c`, `directory.c` helpers, `mkdir.c`, `dirRead.c`, `remove.c`)
- ☐ Quynh did Milestone 2 Day 4 (`initPaths`, Makefile, shell switches)
- ☐ Run 1 and run 2 of the Milestone 2 test work

**Target: Thu Nov 12.** Day 1 of Milestone 3 starts after this.

---

## 1. What must work

**Shell commands** (we turn them ON in `fsshell.c`):
- ☐ `touch` (make an empty file)
- ☐ `cat` (show a file)
- ☐ `cp` (copy a file inside our file system)
- ☐ `cp2fs` (copy a Linux file into our file system)
- ☐ `cp2l` (copy our file out to Linux)
- ☐ `mv` (move or rename a file or folder)
- ☐ `rm` on a file (Quynh's `fs_delete` from Milestone 2)
- ☐ All Milestone 2 commands still work: `ls`, `ls -l`, `md`, `cd`, `pwd`, `rm`
- ☐ After the shell restarts, all files and their data are still there.

**Functions:**

| Function | What it does | Used by | Who |
|---|---|---|---|
| `b_open` | Find or create a file, get it ready to read/write | all file commands | Quynh |
| `b_seek` | Move the position in the file | (not used by the shell, but required) | Trieu |
| `b_close` | Write the last data and the new size to disk | all file commands | Quynh |
| `b_read` | Read bytes from a file | `cat`, `cp`, `cp2l` | Mike |
| `b_write` | Write bytes to a file, get new blocks when it grows | `cp`, `cp2fs` | Trieu |
| `fs_move` + `cmd_mv` | Move or rename | `mv` | Jake (+ Quynh wires the shell) |

**Writeup:** the final PDF needs everything in section 11 (approach for each milestone, description, **command table**, issues, function details, **who worked on what down to each function**, **screenshots of every command**). Plus the volume file and each person's **individual report**.

---

## 2. Key words

| Term | Simple meaning |
|---|---|
| **File** | A directory entry **without** the directory flag. Its data lives in a chain of blocks in the FAT. |
| **fd** (file descriptor) | A small number (0 – 19) that `b_open` gives back. The shell uses it for every later call. |
| **FCB** (file control block) | One slot in `fcbArray`. It remembers everything about one open file: where it is, its size, the position, the buffer. `fd` = the slot number. |
| **Position** | Where the next read or write happens, in bytes from the start of the file. |
| **Buffer** | One block of the file kept in memory, so we do not call `LBAread`/`LBAwrite` for every few bytes. |
| **Dirty** | The buffer has changes that are not on disk yet. They must be written before we use another block or close. |
| **Flags** | How the file is opened. `O_RDONLY`, `O_WRONLY`, `O_RDWR` = what we may do. `O_CREAT` = make it if missing. `O_TRUNC` = empty it. `O_APPEND` = start at the end. |
| **Chain** | The list of a file's blocks, linked in the FAT: start block → next → next → -1. |

---

## 3. The big idea

### 3.1 How a file is stored

A file is just **a directory entry + a chain of blocks**. We already have both from Milestones 1 and 2.

Example: `notes.txt`, 1,000 bytes, inside the root:

| Where | What |
|---|---|
| root, entry 2 | name "notes.txt", size **1000**, startBlock **160**, permission 0644 (no directory flag) |
| FAT[160] | 161 (next block) |
| FAT[161] | -1 (the end) |
| block 160 | bytes 0 – 511 |
| block 161 | bytes 512 – 999 (the last 24 bytes of the block are not used) |

- An **empty file** (after `touch`) has size 0 and startBlock **-1**: no blocks at all. (The same rule `remove.c` already uses.)
- To find byte 700: 700 ÷ 512 = block **1** of the file → follow the chain 1 step from 160 → disk block 161, at offset 700 – 512 = 188.

### 3.2 How an open file works

| Call | What happens |
|---|---|
| `b_open("notes.txt", O_RDONLY)` | Find the entry with `ParsePath`. Take a free FCB slot. Copy the start block and the size into it. Get one block of memory for the buffer. Give back the slot number (fd). |
| `b_read(fd, buf, 200)` | Read 200 bytes at the position, through the buffer. Move the position. |
| `b_write(fd, buf, 200)` | Put 200 bytes into the buffer at the position. When a block is full, write it and go to the next block (get a new one if needed). |
| `b_close(fd)` | Write the dirty buffer. Write the new size and start block into the directory entry. Save the FAT. Free the slot. |

### 3.3 Our 6 design rules

1. **The FCB keeps where the entry is, not a copy of the folder.** It saves a copy of the parent's **"." entry** (`parentDot`) and the entry's **index**. `LoadDirectory` takes an entry, so `b_close` does `LoadDirectory(&parentDot)` and gets the folder fresh from the disk. It is never out of date.
2. **The buffer holds one block of the file, and the FCB says which one** (`bufFileBlock` = block 0, 1, 2 … of the file, `bufDiskBlock` = its disk block). `b_read` and `b_write` check this before they use the buffer. Then reading and writing the same file (`O_RDWR`) and `b_seek` work with no special code.
3. **Write the buffer before you leave it.** If it is dirty, call `flushBuffer` before loading another block, before reading straight from the disk, and in `b_close`.
4. **Files grow one block at a time.** When `b_write` needs a block the file does not have, it asks Mike's `extendChain` for one more (or `allocBlocks(1)` for the first block). Simple, and the FAT always stays a correct chain.
5. **Save at the end, not after every block.** `b_write` marks `fatChanged`. `b_close` writes the entry and calls `saveVolume` once. (`O_TRUNC` in `b_open` releases blocks, so it saves at once, like `rm`.)
6. **Bad input never crashes.** The shell does not check what `b_open` gives back (for example `cp nothere x` still calls `b_read(-1, …)`). So every function first checks the fd and the count, and gives back **-1**.

---

## 4. Who does what

| Member | Files (only this person edits them) | Functions | Test with | Writeup | Reviewer |
|---|---|---|---|---|---|
| **Quynh** | `fcb.h` (new), `b_io.c`, `b_io.h` (header only), `fsshell.c`, `Makefile` | `getOpenFCB`, `flushBuffer`, `canRead`, `canWrite`, `b_open`, `b_close`, `cmd_mv`, connecting everything | `make run` | Approach M3 (open/close + flags) | Trieu |
| **Mike** | `freeSpace.h`, `freeSpace.c` (add), `bRead.c` (new), `tests/testFreeSpace.c` | `blockAtIndex`, `lastBlockInChain`, `extendChain`, `b_read` | `tests/testFreeSpace.c`, then `cat`/`cp2l` in the shell | Approach M3 (reading), **the Shell Command Functionality table** | Jake |
| **Trieu** | `bWrite.c` (new) | `b_write`, `b_seek` | `cp2fs`/`cp` in the shell | Approach M3 (writing + seek), **full test run + valgrind**, Issues table, **Description** section | Quynh |
| **Jake** | `mfs.h` (only the `fs_move` line), `move.c` (new), `tests/testMove.c` | `isInside`, `fs_move` | `tests/testMove.c`, then `mv` in the shell | Approach M3 (mv), **screenshots of every command**, **put the PDF together + submit** | Mike |

**Who needs whom:**

| Order | Part | Who needs it |
|---|---|---|
| 1st (Day 1) | Quynh's `fcb.h`, Mike's new prototypes in `freeSpace.h`, Jake's line in `mfs.h` | everyone can compile alone after this |
| 2nd (Day 2) | Quynh's `getOpenFCB` + `flushBuffer`; Mike's 3 FAT helpers | Mike's `b_read`, Trieu's `b_write` |
| 3rd (Day 3) | `b_open` / `b_close` (Quynh), `b_read` (Mike), `b_write` + `b_seek` (Trieu), `fs_move` (Jake) | the shell |
| 4th (Day 4) | Quynh: Makefile + shell switches + `cmd_mv` | the full test |

- **Day 1, everyone commits their `.h` change first**, like Milestone 2.
- Mike's FAT helpers and Jake's `fs_move` can be **tested alone** (they do not need any open file).
- `b_read` and `b_write` need `b_open`, so they are tested in the shell on Day 4. Until then, compile alone with `gcc -c -Wall -I.`.

---

## 5. Rules for everyone

Same as Milestone 2, plus 2 new ones (8 and 9):

1. Edit **only your own files**. `git pull` before you start. Add only your own files by name (never `git add .`).
2. **After you change a `.h` file, run `make clean`, then `make`.**
3. Your code must build before every commit. Until Day 4, compile your new file alone: `gcc -c -Wall -I. yourfile.c && rm yourfile.o`.
4. One commit per function (or small group). **3 or more commits each.**
5. Commit only `.c`, `.h` and `Makefile`. Do **not** commit `fsshell`, `.o` files, test programs or `TestVolume`. After testing, run `git restore SampleVolume fsshell`. (Exception: the **final** `SampleVolume` on Day 7, because the professor wants the volume file.)
6. Every new file starts with **your** class header on line 1 (`CSC-415-03 Fall 2026`, name, ID, GitHub name, `Combo Number 4`, `Basic File System`, file name, 1 – 3 lines).
7. Simple, clear code. Short comments that explain **why**. Lines around 80 characters, **never over 100** (professor's rule).
8. **Check the fd and the count first** in every `b_` function (rule 6 in section 3.3).
9. **Free what you malloc.** Every `LoadDirectory` result is freed with `free`, and every `ParsePath` parent with **`freeParent`** (never plain `free`: it can be the loaded root or cwd). We check with `make vrun` on Day 5.
10. **Use the Milestone 2 rules for paths:** give `ParsePath` a **copy** of the path (it changes the string). `ParsePath` returns 0 = ok, -1 = invalid, -2 = not found. Index **-2** means the path is just "/", so check the index **before** you use `lastElementName` (it is NULL then). After every `writeDirectory`, call `refreshLoadedDirs`.

---

## 6. Quynh: the FCB, `b_open`, `b_close` + connecting everything

**The idea:** `b_open` is like **checking out a library book**. You find it on the shelf (the directory), get a slip (the FCB slot = fd) that remembers which book and which page you are on. `b_close` is returning it: you write down the new page count (size) on the shelf card and free the slip.

**Files:** `fcb.h` (new), `b_io.c` (fill in), `b_io.h` (fill in the header only), `fsshell.c`, `Makefile`

### Step 1 (Day 1): the contract in `fcb.h`
Include `b_io.h`. Move the professor's `b_fcb` struct here (so Mike and Trieu can use it) and add our fields:

| Field | Meaning |
|---|---|
| `buf` | one block of memory. **NULL = this slot is free** (the professor's rule) |
| `bufFileBlock` | which block of the file is in `buf` (0, 1, 2 …), **-1 = none** |
| `bufDiskBlock` | the disk block number of that block (to write it back) |
| `dirty` | 1 = `buf` has changes not on disk yet |
| `position` | the next byte to read or write |
| `fileSize` | the size in bytes |
| `startBlock` | the first block of the file, **-1 = no blocks yet** |
| `parentDot`, `entryIndex` | a copy of the parent's "." entry, and where the file's entry is in it (rule 1) |
| `flags` | the flags from `b_open` |
| `fatChanged` | 1 = `b_write` took new blocks, so `b_close` must save |

Also: `MAXFCBS 20`, `extern b_fcb fcbArray[MAXFCBS];` and 4 prototypes:

| Function | Gets | Gives back |
|---|---|---|
| `getOpenFCB` | an fd | a pointer to its FCB, or **NULL** if the fd is out of range or not open |
| `flushBuffer` | an FCB | 0 (writes `buf` to `bufDiskBlock` only if dirty, then dirty = 0), -1 on error |
| `canRead` | an FCB | 1 if opened with `O_RDONLY` or `O_RDWR` (use `flags & O_ACCMODE`) |
| `canWrite` | an FCB | 1 if opened with `O_WRONLY` or `O_RDWR` |

Also in `b_io.c` on Day 1: fill in the class header, **delete the professor's empty `b_read`, `b_write` and `b_seek`** (add a comment: "b_read is in bRead.c, b_write and b_seek are in bWrite.c"), and include `fcb.h`. Do the same header fix in `b_io.h`.

### Step 2 (Day 2): the 4 small helpers in `b_io.c`
Write `getOpenFCB`, `flushBuffer`, `canRead`, `canWrite`. **Push them by evening.** Mike and Trieu use them.

**Also make `b_init` and `b_getFCB` work with `fcbArray` from `fcb.h`.** Keep them in `b_io.c`.

### Step 3 (Day 3): `b_open`, `b_close`
Include `mfs.h`, `vcb.h`, `freeSpace.h`, `directory.h`, `path.h`, `fsInit.h`, `fcb.h`.

**`b_open`, in this order:**

| Step | What to do | If it fails |
|---|---|---|
| 1 | `b_init` if needed. Filename must not be NULL or empty. `b_getFCB` for a free slot. | -1 (print "Too many open files" if no slot) |
| 2 | Copy the filename, `ParsePath` the copy. | not 0 → print "… bad path", -1 |
| 3 | **"/"** (index -2), or **found but a directory** (`isDEaDir`) → print "… is a directory", `freeParent`, -1. (Check -2 first.) | |
| 4 | **Not found** (index -1): if no `O_CREAT`, print "… not found", `freeParent`, -1. With `O_CREAT`: the name must be shorter than 36; `findFreeEntry` in the parent (full → print, `freeParent`, -1). Fill the entry: name = `lastElementName`, size 0, startBlock **-1**, permission **0644**, created = modified = now. `writeDirectory` the parent, then `refreshLoadedDirs`. (No FAT change, so no `saveVolume`.) | |
| 5 | `O_TRUNC` **and** `canWrite`, and the file has blocks: `releaseBlocks`, startBlock -1, size 0, modified now. `writeDirectory`, `refreshLoadedDirs`, then `saveVolume` (the FAT changed). | |
| 6 | Fill the FCB: `buf = malloc(blockSize)`, bufFileBlock -1, dirty 0, start block and size from the entry, parentDot = a copy of `parent[0]`, entryIndex, flags. Position = **size** with `O_APPEND`, else 0. | malloc fails → `freeParent`, -1 |
| 7 | `freeParent`. Give back the fd. | |

**`b_close`:**
1. `getOpenFCB` (NULL → -1).
2. `flushBuffer`.
3. If `canWrite`: `LoadDirectory(&parentDot)`, set the entry's size, startBlock, modified = now. `writeDirectory`, then `refreshLoadedDirs`. Free the folder. If `fatChanged`: `saveVolume`. (So `touch` on an old file updates its time, like Linux.)
4. Free `buf` and set it to **NULL** (the slot is free again). Give back 0.

### Step 4 (Day 4): connect it all (after everyone has pushed)
1. **Makefile:** add `bRead.o bWrite.o move.o` to `ADDOBJ` (`b_io.o` is already there).
2. **fsshell.c:**
   - Turn on `CMDTOUCH_ON`, `CMDCAT_ON`, `CMDCP_ON`, `CMDCP2FS_ON`, `CMDCP2L_ON`, `CMDMV_ON` (all 11 commands ON).
   - **`cmd_mv`:** replace the TODO: check `argcnt == 3` (else print "Usage: mv source dest", -1), then give back `fs_move(argvec[1], argvec[2])`.
   - **`cmd_cp` fix:** with only one name, `dest = src`, so `cp a` opens the same file with `O_TRUNC` and **erases it**. Change case 2 to print the usage and give back -1.
   - Fill in the class header (the professor's README says every file needs it).
3. `make clean`, `make`, then the full test (section 10).

### Done when
- ☐ `make` has no warnings, and the command table at start shows **all 11 commands ON**.
- ☐ Run 1 and run 2 in section 10 give the results shown.

**Quynh's commits (example):** `Add fcb.h with the FCB and helper prototypes` → `Add getOpenFCB, flushBuffer, canRead, canWrite` → `Add b_open with O_CREAT and O_TRUNC` → `Add b_close` → `Add the new files to the Makefile` → `Add cmd_mv, fix cp with one name, turn on all commands`

---

## 7. Mike: FAT helpers + `b_read`

**The idea:** `b_read` is like **reading a long book through a small window**. You can only see one page (block) at a time. You copy what you need from this page; if you need more, you turn the page (follow the FAT to the next block). You stop at the end of the book (the file size), not at the end of the page.

**Files:** `freeSpace.h`, `freeSpace.c` (add), `bRead.c` (new), `tests/testFreeSpace.c` (add)

### Step 1 (Day 1): the contract in `freeSpace.h`
Add 3 prototypes before `#endif`:

| Function | Gets | Gives back |
|---|---|---|
| `blockAtIndex` | a start block, n | the disk block that is **n steps** down the chain (n = 0 → the start block), or -1 if the chain is shorter |
| `lastBlockInChain` | a start block | the last block (the one with -1 in the FAT), or -1 for a bad start |
| `extendChain` | the last block of a chain, a count | `allocBlocks(count)`, then link: FAT[last] = the new first block. Gives back the new first block, or -1 (nothing changed) |

### Step 2 (Day 2): the helpers in `freeSpace.c` + test alone
Use `nextBlock` for walking. Check the range every step (the same safety as `allocBlocks`). **Push them by evening.** Trieu needs them.

**Test** (`tests/testFreeSpace.c`, add a part): on a new TestVolume, `allocBlocks(3)` → 154 → 155 → 156.

| Check | You should see |
|---|---|
| `blockAtIndex(154, 0)` / `(154, 2)` / `(154, 3)` | 154 / 156 / -1 |
| `lastBlockInChain(154)` | 156 |
| `extendChain(156, 2)` | 157, and now `blockAtIndex(154, 4)` = 158 and `lastBlockInChain(154)` = 158 |
| free count | 5 fewer than at the start |
| `releaseBlocks(154)` | all 5 back, free count as at the start |

### Step 3 (Day 3): `b_read` in `bRead.c`
Include `fcb.h`, `vcb.h`, `freeSpace.h`, `fsLow.h`. Use the professor's 3-part idea (the big comment in the old `b_read`):

| Step | What to do |
|---|---|
| 0 | `getOpenFCB` (NULL → -1). Not `canRead` → -1. count < 0 → -1. |
| 1 | **Do not read past the end:** if count > size – position, count = size – position. count 0 → give back 0. |
| 2 | `flushBuffer` (rule 3: the disk must be up to date before we read it). |
| 3 | **Part 1:** if `bufFileBlock` = position ÷ blockSize, copy what we need from `buf` (from position % blockSize). Move the position. |
| 4 | **Part 2:** while we still need **a whole block** and the position is at a block start: `blockAtIndex`, `LBAread` **straight into the caller's buffer**, move the position. (One block per `LBAread`, because a chain may not be next to each other.) |
| 5 | **Part 3:** if bytes are still needed: load the block for the position into `buf` (`blockAtIndex`, `LBAread`), set `bufFileBlock`/`bufDiskBlock`, copy the rest. |
| 6 | Give back how many bytes were copied. |

`cat` stops when `b_read` gives back less than it asked for, so step 1 matters.

### Step 4 (Day 6): the Shell Command Functionality table
For all 13 commands, write **Working**, **Not working** or **Partly working**, using Jake's screenshots and the Day 5 test. Put the limits from section 12 in the Limitations column.

### Done when
- ☐ Your test gives the results above.
- ☐ On Day 4, `cat` shows the full file, and `cp2l` + Linux `diff` says the files are the same (section 10).

**Mike's commits (example):** `Add the Milestone 3 prototypes to freeSpace.h` → `Add blockAtIndex, lastBlockInChain, extendChain` → `Test the new FAT helpers` → `Add b_read in bRead.c`

---

## 8. Trieu: `b_write` + `b_seek`

**The idea:** `b_write` is like **filling a notebook one page at a time**. You write on the current page (the buffer). When the page is full, you put it on the shelf (write the block) and take the next page. If the notebook has no more pages, you add a new one (get a block from the free list).

**Files:** `bWrite.c` (new)

### Step 1 (Day 1): read the contracts
No `.h` for you this time. Read `fcb.h` (Quynh) and the new `freeSpace.h` lines (Mike) when they are pushed, and finish `fs_mkdir` from Milestone 2 if it is not done.

### Step 2 (Day 2 – 3): `b_write` in `bWrite.c`
Include `fcb.h`, `vcb.h`, `freeSpace.h`, `fsLow.h`.

| Step | What to do |
|---|---|
| 0 | `getOpenFCB` (NULL → -1). Not `canWrite` → -1. count < 0 → -1. count 0 → 0. |
| 1 | Loop while bytes are left: n = position ÷ blockSize, offset = position % blockSize. |
| 2 | **Get block n into the buffer** (skip if `bufFileBlock` is already n): `flushBuffer`. `blockAtIndex(startBlock, n)`. If the file **has** that block: `LBAread` it. If **not** (-1): get a new one: no blocks yet → `allocBlocks(1)` and set `startBlock`; else → `extendChain(lastBlockInChain(startBlock), 1)`. No space → **stop** and give back the bytes written so far. Fill the new `buf` with zeros. Set `fatChanged = 1`. Set `bufFileBlock = n`, `bufDiskBlock`. |
| 3 | Copy as much as fits: the smaller of (blockSize – offset) and the bytes left. `dirty = 1`. Move the position. If position > fileSize, fileSize = position. |
| 4 | After the loop, give back how many bytes were written. |

**Why read the block first (step 2)?** If we write into the middle of an old block, the rest of that block must stay the same.
**Why not `saveVolume` here?** `cp2fs` calls `b_write` many times. `b_close` saves once at the end (rule 5).

### Step 3 (Day 3): `b_seek` in `bWrite.c`
`b_seek` uses the same position and size as `b_write`, so it lives in the same file.

| Step | What to do |
|---|---|
| 1 | `getOpenFCB` (NULL → -1). |
| 2 | New position by `whence`: `SEEK_SET` = offset, `SEEK_CUR` = position + offset, `SEEK_END` = size + offset. Anything else → -1. |
| 3 | It must be **0 to size** (we do not make "holes"), else -1. |
| 4 | Set the position and give it back. |

**Why no buffer work?** `b_read` and `b_write` always check `bufFileBlock` against the position (rule 2), so they load the right block by themselves.

### Step 4 (Day 5): lead the full test
Run section 10 (run 1, run 2 and the edge cases) and `make vrun`. Write down every problem in the **Issues and Resolutions** table.

### Step 5 (Day 6): the Description section
Update the writeup's **Description** for the finished file system: add files (a directory entry + a FAT chain, empty file = startBlock -1), the FCB and the one-block buffer, and how data stays after a restart. Keep the Milestone 1 volume layout part.

### Done when
- ☐ On Day 4, `cp2fs` of a big file (more than 1 block) + `cp2l` back + Linux `diff` says the same (section 10).
- ☐ `ls -l` shows the right size.
- ☐ `b_seek` gives back -1 for a bad fd, a bad `whence`, or a position outside 0 to size.

**Trieu's commits (example):** `Add b_write in bWrite.c` → `Get new blocks when the file grows` → `Add b_seek` → (fixes from testing)

---

## 9. Jake: `mv`

**The idea:** `mv` is like **moving a card from one box to another**. The data (the blocks) does not move at all. Only the directory entry moves: copy it into the new folder, then clear it in the old folder. Renaming in the same folder is just changing the name on the card.

**Files:** `mfs.h` (only one new line), `move.c` (new), `tests/testMove.c`

### Step 1 (Day 1): the contract in `mfs.h`
Add one prototype under the others: `int fs_move(const char *src, const char *dest);` (0 = ok, -1 = error). The shell only includes `mfs.h`, so it must be here.

### Step 2 (Day 2): `isInside` (private helper in `move.c`)
Gets a folder start block A and a folder start block B. Walk **up** from A using each folder's ".." (`LoadDirectory(&dir[1])`, then free the old one) until the root. 1 if we meet B on the way (or A = B), else 0. We need it twice in step 3.

### Step 3 (Day 2 – 3): `fs_move` in `move.c`
Include `mfs.h`, `vcb.h`, `directory.h`, `path.h`.

| Step | What to do | If it fails, print |
|---|---|---|
| 1 | Copy the source, `ParsePath` the copy. It must be found, and not "/" (index -2, check first), "." or "..". | "mv: cannot move …" |
| 2 | If the source is a folder: the **cwd must not be inside it** (`isInside(getCwdStart(), src)`), or `pwd` would show a path that no longer exists. | "mv: cannot move the current directory" |
| 3 | Copy the dest, `ParsePath` the copy (a **second** `ppinfo`). Found **and a folder** → move **into** it, keep the old name. **Not found** → its parent is the target, the last name is the new name. Found **and a file** → error (we do not overwrite). A new name must be shorter than 36. | "mv: … already exists" / "mv: bad path …" |
| 4 | If the source is a folder: the target must **not be inside the source** (`md a`, `mv a a/b` would lose `a`). | "mv: cannot move a folder into itself" |
| 5 | **Same folder** (same start block) → just copy the new name into the entry, `writeDirectory` once. Done. | |
| 6 | **Other folder** → `findFreeEntry` in the target (full → error). Copy the entry there with the new name. If it is a folder, `LoadDirectory` it and set its ".." to the target (copy the target's ".", name it ".."), write it. `writeDirectory` the target **first**. Then clear the old entry (zeros, startBlock -1) and `writeDirectory` the old parent. Set both "." modified times. After each `writeDirectory`, `refreshLoadedDirs`. | "mv: the directory is full" |
| 7 | `freeParent` both `ppinfo`s (and `free` what you loaded). No FAT change, so no `saveVolume`. Give back 0. | |

**Why write the target first (step 6)?** If something fails in the middle, we have the entry twice (easy to fix) instead of zero times (the file is lost).

### Step 4 (Day 3): test alone (`tests/testMove.c`)
Same setup as `testPath.c` (VCB in memory, root 154, "a" 155, "b" 156 inside "a"), plus a fake file "f.txt" in the root.

| Call | You should see |
|---|---|
| `fs_move("f.txt", "g.txt")` | 0; the root has "g.txt" at the same index |
| `fs_move("g.txt", "a")` | 0; "g.txt" is now in "a", the root entry is free |
| `fs_move("a/b", "/")` | 0; "b" is in the root, and b's ".." = 154 |
| `fs_move("a", "a/x")` | -1 (into itself) |
| `fs_move("nothere", "a")` | -1 |
| `fs_move(".", "a")` | -1 |

### Step 5 (Day 6): screenshots
Run every command in the shell (all 13: `ls`, `cp`, `mv`, `md`, `rm`, `touch`, `cat`, `cp2l`, `cp2fs`, `cd`, `pwd`, `history`, `help`) and take a screenshot of each. Share them with Mike for the command table.

### Step 6 (Day 7 – Fri Nov 20): put the PDF together + submit
1. Check that every part in section 11 is filled in. Fix the cover page **submission date**.
2. Ask everyone to read it once. Export it as a **PDF**.
3. Commit the PDF and the **final** `SampleVolume` (one time only, rule 5).
4. Submit in **Canvas**: the PDF + the GitHub URL. **Only one person submits**, so tell the team when it is done.

**Jake's commits (example):** `Add fs_move to mfs.h` → `Add isInside in move.c` → `Add fs_move` → `Add tests/testMove.c`

---

## 10. Test everything together (Day 4 – 5)

`git pull` → `make clean` → `make` (no warnings) → delete `SampleVolume` → `make run`.

### Run 1: a new volume

| You type | You should see |
|---|---|
| (start) | "Formatting now." and **all 11 commands ON** |
| `touch a.txt`, then `ls -l` | `-  0  a.txt` |
| `cp2fs fsshell.c big.c` | nothing (fsshell.c is about 20 KB = about 40 blocks, so this tests the chain) |
| `ls -l` | big.c with the same size as Linux `ls -l fsshell.c` |
| `cat big.c` | the whole file, the last line too |
| `cp big.c copy.c`, then `ls -l` | copy.c, same size |
| `cp2l copy.c out.c` | nothing |
| (in another terminal) `diff fsshell.c out.c` | **no output** = the same file ✅ |
| `md docs`, `mv copy.c docs`, `ls docs` | copy.c |
| `mv a.txt b.txt`, `ls` | b.txt (no a.txt) |
| `mv docs pics`, `ls` | pics |
| `rm b.txt`, `ls` | no b.txt |
| `exit` | System exiting |

### Run 2: is it still there?
`make run` again (do **not** delete `SampleVolume`).

| You type | You should see |
|---|---|
| (start) | "already formatted. Loading it." |
| `ls -l` | big.c (same size), pics |
| `cat big.c` | the whole file again |
| `cd pics`, `cat copy.c` | the whole file |
| `cd /`, `rm big.c`, `rm pics/copy.c`, `rm pics` | nothing (all worked) |
| `ls` | (empty) |
| `exit` | System exiting |

### Edge cases (must not crash)
`cat nothere` ("Failed to open"), `cp nothere x`, `cp2fs nothere.txt x`, `cat pics` (a folder), `mv pics pics/x`, `mv . x`, `cp a` (one name: usage), a 7th file in one folder ("directory is full").

### Memory check (Trieu)
`make vrun`, do `cp2fs`, `cat`, `cp`, `mv`, `rm`, then `exit`. Look for **"definitely lost: 0 bytes"**.

---

## 11. The final writeup + submission

The professor's list → who does it:

| Required | Who | Notes |
|---|---|---|
| GitHub link, cover page | Jake | already there; fix the **submission date** |
| **Approach Milestone 3** | everyone, one short paragraph | the plan before coding, like M1 and M2 |
| Approach Milestone 2 | everyone | check it is filled in |
| Description of the file system | Trieu | add files: entry + FAT chain, the FCB, the buffer |
| **Shell command table** | Mike | from Jake's screenshots and the Day 5 test; limitations from section 12 |
| Issues and Resolutions | Trieu (everyone adds their own) | real problems only |
| Function Descriptions | everyone adds their own rows | 4 columns: purpose, how it works, notes |
| **Who worked on what (each function)** | everyone adds their own rows | M2 and M3 rows too |
| **Screenshots of every command** | Jake | all 13 commands |
| Volume file (10 MB or less) | Jake | ours is 10,000,000 bytes. Commit the **final** `SampleVolume` once on Day 7 |
| PDF in GitHub **and** Canvas, Git URL in Canvas | Jake | **only one person submits** |
| Individual report (form) | **each person** | not part of the group PDF |

**Approach Milestone 3 paragraphs:**

| Who | Write about |
|---|---|
| **Quynh** | The FCB, how `b_open` handles each flag, why the FCB keeps the parent's start block, why `b_close` saves once, connecting the shell |
| **Mike** | The 3-part read, why we stop at the file size, the FAT helpers |
| **Trieu** | How a file grows one block at a time, why we read a block before writing into the middle of it, why `b_seek` stays between 0 and the size |
| **Jake** | How `mv` moves only the entry (not the data), the "into itself" and cwd checks, why the target is written first |

---

## 12. Known limits (for the command table)

| Limit | Why |
|---|---|
| A folder holds only 6 items | One folder = 1 block (Milestone 2 rule 4). We kept it, so we could finish the file functions well. |
| `ls -l otherfolder` shows wrong sizes | The professor's shell calls `fs_stat` with only the name. `cd` into the folder first. |
| `cp` into a folder needs the full name | `cp a.txt docs` fails; use `cp a.txt docs/a.txt`. (`mv a.txt docs` works.) |
| `mv` does not overwrite an existing file | We chose safety: it says "already exists". |
| `b_seek` cannot go past the end | No "holes" in files. |
| The same file opened twice at once | Not supported (the shell never does this). |
| File names: 35 characters max | The directory entry name is 36 bytes. |

---

## 13. Schedule

| Day | Quynh | Mike | Trieu | Jake |
|---|---|---|---|---|
| **Thu Nov 12** | Milestone 2 Day 4 + full M2 test | M2 done | M2 done | M2 done |
| **Day 1 Fri Nov 13** | `fcb.h` + `b_io.c` cleanup | `freeSpace.h` prototypes | read the contracts | `mfs.h` line |
| **Day 2 Sat Nov 14** | 4 helpers, **push by evening** | 3 FAT helpers + test, **push by evening** | start `b_write` | `isInside` + start `fs_move` |
| **Day 3 Sun Nov 15** | `b_open`, `b_close` | `b_read` | `b_write` + `b_seek` | `fs_move` + `testMove.c` |
| **Day 4 Mon Nov 16** | Makefile + shell (morning), then run 1 | test `cat`/`cp2l` | test `cp2fs`/`cp` | test `mv` |
| **Day 5 Tue Nov 17** | fix bugs together at our meeting: run 1, run 2, edge cases, `make vrun`; reviewers read the code | | | |
| **Day 6 Wed Nov 18** | writeup: own rows | **command table**, own rows | Issues table, **Description**, own rows | **screenshots** |
| **Day 7 Thu Nov 19** | Approach M3 paragraphs (everyone). Jake puts the PDF together. Everyone reads it once. | | | |
| **Fri Nov 20** | **Jake submits** (GitHub: code + PDF + `SampleVolume`; Canvas: PDF + Git URL) | | | |
| **Nov 20 – 22** | each person: **individual report** | | | |

Buffer: if we are late, we still have Nov 21 – 26. But only **1 attempt**, so we submit only when the final checklist is all ✅.

---

## 14. Final checklist (before we submit)

- ☐ `make clean` then `make`: no warnings
- ☐ The command table at start shows **all 11 commands ON**
- ☐ Run 1 and run 2 in section 10 work, `diff` shows no output
- ☐ The edge cases do not crash
- ☐ `make vrun`: "definitely lost: 0 bytes"
- ☐ Every `.c` and `.h` file has our class header on line 1 (also `b_io.c`, `b_io.h`, `fsshell.c`)
- ☐ No line over 100 characters
- ☐ Each person has 3 or more commits on their own files
- ☐ No `fsshell`, `.o`, test programs or `TestVolume` committed; the **final** `SampleVolume` is committed
- ☐ The PDF has every part in section 11, and the GitHub link
- ☐ The PDF is in GitHub **and** Canvas, with the Git URL in Canvas
- ☐ Each person submitted their individual report
