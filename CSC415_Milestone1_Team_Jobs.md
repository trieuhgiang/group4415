# Milestone 1: Team Jobs and Instructions ("Formatted Volume")

**Team:** Jake, Mike, Quynh, Trieu
**Design:** our approved `File System Design.docx` (FAT, VCB in block 0, 64-byte directory entries)
**Follows:** the professor's *Steps for milestone 1* (steps 1a to 4j)
**Due:** Wed Nov 4, 2026, 11:59 pm. There is **one attempt only**, so we submit on **Tue Nov 3**. (Check the date on the class page.)
**Submits:** Jake

---

## 1. What we must hand in

**Code:**
- ☐ The **VCB** is written in block 0.
- ☐ The **free space** is set up, and **allocate** works. (Release is not needed yet.)
- ☐ The **root directory** is written, with **"."** and **".."**.
- ☐ The 1st run formats the volume. The 2nd run finds the signature and does **not** format again.

**PDF** (use the professor's **new write-up template** from the class page):
- ☐ **P0:** A cover page with the team name, the 4 names and student IDs, and the GitHub name.
- ☐ **P1:** A hexdump of the VCB, the free space, and the complete root directory.
- ☐ **P2:** The VCB structure. **P3:** The free-space structure. **P4:** The directory system.
- ☐ **P5:** A table of who did what. **P6:** How we worked together. **P7:** Our issues and how we fixed them.

---

## 2. Key words

| Term | Simple meaning |
|---|---|
| **Volume** | The file `SampleVolume`. It acts as our hard disk. |
| **Block** | One piece of the volume: 512 bytes. We read and write whole blocks. |
| **LBA** | Logical Block Address: the number of a block, starting at 0. `LBAread` and `LBAwrite` use it. |
| **VCB** | Volume Control Block. Block 0. It holds the facts about the whole volume. |
| **Signature** | A known value in the VCB (`"FAT415"`). If we find it, the volume is already formatted. |
| **FAT** | File Allocation Table. One number for every block. The number at block *i* = the block that comes after block *i*. |
| **Chain** | Blocks linked by the FAT, like 154 → 155 → 156 → end. |
| **Free list** | One chain that holds all the free blocks. |
| **Directory entry** | A 64-byte record about one file or folder. **All the metadata of a file lives here.** |
| **Contract** | The function prototypes in a `.h` file. Others can write their code against them before your `.c` file is done. |

---

## 3. Our volume in numbers

The shell runs with `SampleVolume 10000000 512`, so the file system gets **19,531 blocks**.

| Part | Our blocks | Hexdump blocks | Notes |
|---|---|---|---|
| VCB | 0 | 1 | always block 0 |
| FAT | 1 – 153 | 2 – 154 | 19,531 × 4 bytes = 78,124 bytes → 153 blocks (rounded up) |
| Root directory | 154 | 155 | 1 block = 8 entries |
| Free list | 155 – 19,530 | 156 – 19,531 | **19,376 free blocks** after the format |

- **Hexdump block = our block + 1.** Block 0 of the file is fsLow's partition table.
- **FAT values:** a number *n* = "next is block *n*". **-1** = end of a chain. **-2** = reserved (the VCB and the FAT blocks).

---

## 4. Who does what

| Member | Files (only this person edits them) | Professor's steps | Test alone with | PDF part | Reviewer |
|---|---|---|---|---|---|
| **Jake** | `vcb.h`, `vcb.c`, `tests/testVCB.c` | 1b, 1c, 2a, 2e | `tests/testVCB.c` | P1 (VCB dump), P2, **submit** | Quynh |
| **Mike** | `freeSpace.h`, `freeSpace.c`, `tests/testFreeSpace.c` | 3a – 3e, allocate | `tests/testFreeSpace.c` | P1 (FAT dump), P3 | Jake |
| **Trieu** | `directory.h`, `directory.c`, `tests/testDirectory.c` | 4a – 4j | `tests/testDirectory.c` | P1 (root dump), P4 | Mike |
| **Quynh** | `fsInit.c`, `Makefile` | 1a – 1d, 2a – 2e (the main flow) | `make run` | P0, P5, P6, the full PDF | Trieu |
| **Everyone** | | | | P7: one or more real issues each | |

**No one waits for anyone:**
- Jake, Mike and Trieu each use **only** the professor's `fsLow` and their own files. Their code **never calls another member's code**.
- Each of them builds and runs their part **alone**, with their own small test program in `tests/`.
- **Quynh's `fsInit.c` is the only file that connects the parts.** It calls everyone's functions and passes the numbers between them.
- Everyone commits their `.h` file (the contract) **on day 1**. After that, nobody waits.

---

## 5. Rules for everyone

1. Edit **only your own files**. Then there are no merge conflicts.
2. Make one commit for each function (or each small group of functions). Make **3 or more commits each**.
3. Your code must build before every commit. If it does not build, do not commit.
4. Commit only `.c`, `.h` and `Makefile`. Do **not** commit `fsshell`, `.o` files, test programs, `SampleVolume` or `TestVolume`.
5. **Only Jake submits** (the GitHub link of our repo + the PDF). **Do not change the repo name.**
6. Tests go in a `tests/` folder. They must be **unobtrusive**: `make` and `make run` work the same as before.
7. Each test program makes its own `TestVolume` file. Never test on `SampleVolume`. Delete `TestVolume` after testing.
8. To build a test alone, compile your test file + your `.c` file + `fsLow.o` (on Apple Silicon, use `fsLowM1.o`), and link the math library (`-lm`).
9. Write simple, clear code: short functions, clear variable names, and comments that explain **why**. Lines should be about 80 characters, never more than 100. In your comments, write which professor's step it is (for example, "Step 3c").
10. Every `.c` and `.h` file starts with the **class header on line 1** (the format is in the README). **Only the person who wrote the file goes in its header.** Do not change the number of `*`.

| Header field | What to write |
|---|---|
| Class | `CSC-415-0# Fall 2026` (put our section number in place of `#`) |
| Name | **your** name only |
| Student IDs | **your** student ID only |
| GitHub-Name | **your** GitHub name |
| Group-Name | our team name |
| Project | `Basic File System` |
| File | the file name |
| Description | 1 – 3 lines: what this file does |

---

## 6. Jake: the VCB

**The idea:** the VCB is like the **cover page of a notebook**. It says how big the notebook is and where each part starts. We read it first, every time the program starts.

**Files:** `vcb.h`, `vcb.c`, `tests/testVCB.c`

### Step 1 (day 1): `vcb.h`, the contract
- The VCB structure, with exactly the 6 fields from our approved design:

| Field | Meaning |
|---|---|
| `totalBlocks` | how many blocks the volume has |
| `freeBlocks` | how many blocks are still free |
| `blockSize` | bytes in one block (512) |
| `freeListHead` | the first block of the free list |
| `rootDirStart` | the first block of the root directory |
| `volumeType` (8 characters) | `"FAT415"`, our signature |

- Two constants: the VCB block number (0) and the signature (`"FAT415"`).
- The prototypes of the 6 functions below.

### Step 2: `vcb.c`, the functions

| Function | What it must do | Professor's step |
|---|---|---|
| `loadVCB` | Get memory for **one whole block** (not only the size of the structure, because `LBAread` always reads 512 bytes). Read block 0 into it. Return 0 if it worked, -1 if not. | 1b |
| `isVolumeFormatted` | Compare the 8 signature bytes with `"FAT415"`. Return 1 if they match, 0 if not. (A new volume is all zeros, so it does not match.) | 1c |
| `initVCB` | Clear the whole block to zeros. Then set `totalBlocks`, `blockSize` and the signature. Leave the other 3 fields for Quynh. | 2a |
| `writeVCB` | Write the block to block 0. Return 0 if it worked, -1 if not. | 2e |
| `getVCB` | Give other files the VCB in memory. | |
| `cleanupVCB` | Free the memory. | |

Print a short error message whenever something fails.

### Step 3: test it alone (`tests/testVCB.c`)
Write a small program that opens its own `TestVolume`, then calls `loadVCB`.
- If the volume is not formatted, it calls `initVCB` and `writeVCB`, then prints "No signature. Writing a new VCB."
- If it is formatted, it prints "Signature found" with `totalBlocks`.

Run it **two times**:

| Run | You should see |
|---|---|
| 1st | No signature. Writing a new VCB. |
| 2nd | Signature found. totalBlocks = 19531 |

### Done when
- ☐ The test gives the 2 results above.
- ☐ After Quynh connects everything, the hexdump of block 1 shows `FAT415` and the values in section 10.

**Jake's commits (example):** `Add vcb.h` → `Add loadVCB` → `Add isVolumeFormatted and initVCB` → `Add writeVCB, getVCB, cleanupVCB` → `Add tests/testVCB.c`

---

## 7. Mike: free space with the FAT

**The idea:** every block gets a small note in the FAT: "after me, go to block ___". All free blocks are linked into one line: the **free list**. To allocate, we **take blocks from the front of the line**.

The lecture names **3 primary free-space functions**: initialize, allocate, and release. Milestone 1 grades only the first two. Release is optional now, but Milestone 2 needs it.

**Files:** `freeSpace.h`, `freeSpace.c`, `tests/testFreeSpace.c`

**Important:** Mike's code does **not** use the VCB. It keeps the free-list start and the free-block count in **its own variables**. Quynh copies them into the VCB.

### Step 1 (day 1): `freeSpace.h`, the contract
- Three constants: the FAT starts at block 1, end of chain = -1, reserved = -2.
- The prototypes of the functions below.

### Step 2: `freeSpace.c`, the functions

| Function | What it must do | Professor's step |
|---|---|---|
| `fatBlockCount` | Blocks needed for the FAT: total blocks × 4 bytes, divided by the block size, **rounded up**. (19,531 blocks → 153) | 3a |
| `initFreeSpace` | For a **new** volume. **(3b)** Get zero-filled memory for the whole FAT (153 blocks). **(3c)** Mark blocks 0 to 153 as -2 (the VCB and the FAT). Link all other blocks into one chain: 154 → 155 → … → 19,530, and the last one = -1. **(3e)** Remember the free-list start (154) and the free count (19,377) in your own variables. **(3d)** Write the FAT to disk: 153 blocks, starting at block 1. Return 1 (where the FAT starts), or -1 on error. | 3b – 3e |
| `allocBlocks` | Take *count* blocks from the **front** of the free list. If *count* is less than 1 or more than the free count: print an error, return -1, and **change nothing**. Otherwise: start at the first free block, walk *count* − 1 steps to the last block you take, make the block after it the new start of the free list, mark your last block as -1 (end of the new chain), lower the free count, and return the first block. | allocate |
| `releaseBlocks` *(optional)* | Give a whole chain back. Walk to its last block and count the blocks. Link that last block to the old start of the free list. The chain's first block becomes the new start. Add the count to the free count. Stop with an error if the chain is longer than the volume (it is broken). | release |
| `loadFreeSpace` | For a volume that is **already formatted**. Read the FAT from disk **one time** and keep it in memory (the professor's note). The free-list start and the free count come in as parameters (Quynh takes them from the VCB). | professor's note |
| `writeFreeSpace` | Write the FAT to disk (153 blocks at block 1). Call it every time the FAT changes. | |
| `nextBlock` | Give back the FAT value of a block (the next block), or -1 if the block number is not in the volume. | |
| `getFreeListHead`, `getFreeBlockCount` | Give back your two numbers, so Quynh can put them in the VCB. | |
| `cleanupFreeSpace` | Free the memory. | |

### Step 3: test it alone (`tests/testFreeSpace.c`)
Write a small program that opens its own `TestVolume` and does these steps, printing the results:

| Step | You should see |
|---|---|
| `initFreeSpace` | free list starts at **154**, free blocks = **19,377** |
| `allocBlocks(3)` | returns **154**. The chain is **154 → 155 → 156** (follow it with `nextBlock`) |
| after that | free list starts at **157**, free blocks = **19,374** |
| `allocBlocks(0)` | **-1** (and an error message) |
| `releaseBlocks(154)` | returns **3**. The free list starts at **154** again, free blocks = **19,377** |

(The test has no root directory, so the free list starts at 154. In the real program, the root takes block 154 first.)

### Done when
- ☐ The test gives the results above.
- ☐ After Quynh connects everything, the FAT hexdump matches section 10.

**Mike's commits (example):** `Add freeSpace.h` → `Add fatBlockCount and initFreeSpace` → `Add allocBlocks` → `Add load/write/next/getters/cleanup` → `Add releaseBlocks` (optional) → `Add tests/testFreeSpace.c`

---

## 8. Trieu: the root directory

**The idea:** a directory is like a **page in a phone book**. Each line (an **entry**) holds one name and where to find it. A new directory has two lines filled in: "." (this directory) and ".." (the parent). All other lines are empty.

**Files:** `directory.h`, `directory.c`, `tests/testDirectory.c`

**One change from the lecture:** the lecture's `createDir` calls allocate *inside*. We moved that one call to Quynh's `fsInit.c`. She allocates the blocks and **gives the start block to Trieu's function**. So Trieu's code does not need Mike's code. The steps are the same; only *who calls allocate* is different.

### Step 1 (day 1): `directory.h`, the contract
- The directory entry structure, with exactly the fields from our approved design (largest first, so the total is **64 bytes** with no padding):

| Field | Size | Meaning |
|---|---|---|
| `created` | 8 bytes | creation time |
| `modified` | 8 bytes | last change time |
| `size` | 4 bytes | size in bytes |
| `startBlock` | 4 bytes | first block (-1 = free entry) |
| `permission` | 4 bytes | rwx bits + bit 9 = "is a directory" |
| `name` | 36 bytes | up to 35 letters + the end mark |

- Four constants: the name length (36), how many entries a new directory gets (**8**), the directory flag (bit 9 = 0x200), and the default permission (0755).
- The prototypes of the 3 functions below.

### Step 2: `directory.c`, the functions

**`directoryBlocksNeeded`** (steps 4b – 4d): entries × 64 bytes, then blocks **rounded up** with the lecture's formula. For 8 entries: 512 bytes → **1 block**. Quynh calls this first, so she knows how many blocks to allocate.

**`createDirectory`**. It gets the number of entries, the parent directory (or nothing for the root), the **start block**, and the block size. It gives back the new directory **in memory**. Do the steps in this order:

| Step | What to do |
|---|---|
| 4b – 4d | Blocks needed (use the function above). Use the whole blocks: bytes = blocks × 512, and actual entries = bytes ÷ 64 = **8**. |
| 4a | Get the memory. Fill it with zeros, so no random bytes go to the disk. |
| 4e | Entries 2 to the end are **free**: empty name and start block **-1**. (Use -1, not 0, because block 0 is the VCB.) |
| 4f | **Not here.** Quynh allocates and gives you the start block. |
| 4g | Entry 0 is **"."**: start block = the given start block, size = actual entries × 64 (= **512**), permission = 0755 + the directory flag (= **0x3ED**), created = modified = now. |
| 4h | Entry 1 is **".."**: if there is no parent (the root), **the directory is its own parent**. Copy the start block, size, permission, created and modified from the parent's entry 0. Name it "..". (These 2 lines are the only difference between the root and any other directory.) |
| 4i | Write it to disk (the function below). If it fails, free the memory and give back nothing. |
| 4j | Give back the directory in memory. (The caller frees it.) |

**`writeDirectory`**: use entry 0 to find the start block and the size. Work out how many blocks that is (rounded up). Write them all with one `LBAwrite` at the start block, like the lecture. Check that all the blocks were written. (This is fine for Milestone 1, because on a new volume a directory's blocks are next to each other. In Milestone 2 we will write one block at a time and follow the FAT chain.)

### Step 3: test it alone (`tests/testDirectory.c`)
You do **not** need Mike's code or a "temporary allocate". Pick **block 154** yourself as the start block.

| Step | You should see |
|---|---|
| `directoryBlocksNeeded(8, 512)` | **1** block |
| `createDirectory` (root, start 154) | entry 0: name ".", start **154**, size **512** |
| | entry 1: name "..", start **154**, size **512** |
| | entry 2: name "", start **-1**, size 0 |
| hexdump of `TestVolume`, block 155 | the same as the root dump in section 10 |

### Done when
- ☐ The test gives the results above.
- ☐ After Quynh connects everything, the hexdump of block 155 shows "." and ".." at block 154, and entries 2 – 7 with start block -1.

**Trieu's commits (example):** `Add directory.h` → `Add directoryBlocksNeeded` → `Add createDirectory: memory and free entries` → `Set . and ..` → `Add writeDirectory` → `Add tests/testDirectory.c`

---

## 9. Quynh: the main flow (connects everything)

**The idea:** `initFileSystem` is our **entry point**: the shell calls it when the program starts. `exitFileSystem` is our **exit point**: the shell calls it when you type `exit`. It is our chance to **flush** (save) everything to disk before the program stops.

**Files:** `fsInit.c`, `Makefile`

Quynh writes her code **against the three `.h` files**. She can start on day 2. Her file can be compiled alone before the other `.c` files are done. She runs the full program after they are in `main`.

### Step 1: set up `fsInit.c`
- Replace the professor's header at the top (it still says "Spring 2024" and is empty) with **your** header.
- Include the three headers: `vcb.h`, `freeSpace.h`, `directory.h`.
- Add one small helper, `copyFreeSpaceToVCB`. It puts Mike's free-list start and free-block count into Jake's VCB.

### Step 2: `initFileSystem`. Do the steps in this order
| Order | What to do | Professor's step |
|---|---|---|
| 1 | Call `loadVCB`. If it fails, return -1. | 1b |
| 2 | If `isVolumeFormatted` says yes: print "The volume is already formatted. Loading it." Check that the VCB's `totalBlocks` and `blockSize` match this volume (if not, print an error and return -1). Then call `loadFreeSpace` with the free-list start and free count **from the VCB**. Stop here. | 1c – 1d |
| 3 | Otherwise print "The volume is not formatted. Formatting now." | 1d |
| 4 | `initVCB` | 2a |
| 5 | `initFreeSpace` (return -1 if it fails) | 2b |
| 6 | Make the root: ask `directoryBlocksNeeded` (8 entries) → `allocBlocks` that many → `createDirectory` with no parent and that start block. Then free the root's memory. (It is on disk now. We load it again in Milestone 2.) | 2c (with 4f) |
| 7 | Put the root's start block in the VCB, then call `copyFreeSpaceToVCB`. | 2d |
| 8 | `writeFreeSpace` again, because the root took a block and the FAT changed. | |
| 9 | `writeVCB` **last**. Then a good signature on disk always means the whole format finished. Return 0. | 2e |

### Step 3: `exitFileSystem`
In this order: `copyFreeSpaceToVCB` → `writeFreeSpace` → `writeVCB` → `cleanupFreeSpace` → `cleanupVCB` → print "System exiting".

### Step 4: `Makefile`
**After** the three `.c` files are in `main`, add `vcb.o`, `freeSpace.o` and `directory.o` to the **`ADDOBJ`** line. (If you do it earlier, `make` fails because the files are missing.)

### Step 5: run it
1. Delete `SampleVolume` (to redo a volume, always delete it by hand).
2. `make`, then `make run`, then type `exit`.
3. `make run` again, then `exit`.

| Run | You should see |
|---|---|
| 1st | Initializing File System with 19531 blocks … / The volume is not formatted. Formatting now. / System exiting |
| 2nd | Initializing File System with 19531 blocks … / The volume is already formatted. Loading it. / System exiting |

Note: the starter Makefile does not watch `.h` files. After a header changes, run `make clean`, then `make`. (`make clean` keeps the professor's `fsLow.o`. Never delete it.)

### Done when
- ☐ The 2 runs give the results above.
- ☐ Everyone checks their hexdump together (section 10).

**Quynh's commits (example):** `initFileSystem: load the VCB and check the signature` → `initFileSystem: format` → `Add copyFreeSpaceToVCB and exitFileSystem` → `Add our three files to the Makefile`

---

## 10. Hexdumps for the PDF (P1)

First make a fresh volume: delete `SampleVolume`, run `make run`, type `exit`. Use `hexdumpl` (Intel) or `hexdumpAS` (Apple Silicon). If it says "permission denied", run `chmod +x Hexdump/hexdumpl`.

| What | Command | Who |
|---|---|---|
| VCB | `./Hexdump/hexdumpl SampleVolume --start 1 --count 1` | Jake |
| FAT, first 2 blocks | `./Hexdump/hexdumpl SampleVolume --start 2 --count 2` | Mike |
| FAT, last block | `./Hexdump/hexdumpl SampleVolume --start 154 --count 1` | Mike |
| FAT, all (save it in the repo) | `./Hexdump/hexdumpl SampleVolume --start 2 --count 153 > fat_full.txt` | Mike |
| Root directory | `./Hexdump/hexdumpl SampleVolume --start 155 --count 1` | Trieu |

**How to read a dump:**
- **2 chunks = 1 block.** Each block prints as 2 chunks of 256 bytes, with an empty line between them.
- Use `--start 1` for the VCB, not 0 (block 0 of the file is the partition table).
- Numbers are **little-endian**: the small byte comes first. `4B 4C 00 00` = 19,531.
- Paste dumps into the PDF as **text in a monospace font** (Courier or Consolas).
- The full FAT is about 80 pages. Put the first 2 FAT blocks and the last one in the PDF, and the full file in the repo. **Ask the professor if this is OK** (one attempt only).

**What you should find:**

| Dump | Look for |
|---|---|
| VCB | totalBlocks 19,531 · freeBlocks 19,376 · blockSize 512 · freeListHead 155 · rootDirStart 154 · `FAT415` |
| FAT | entries 0 – 153 = `FE FF FF FF` (-2) · entry 154 = `FF FF FF FF` (-1, the root) · 155 → 156 → 157 … · entry 19,530 = -1 |
| Root | entry 0 "." and entry 1 ".." both start at 154 (`9A 00 00 00`), size 512, permission `ED 03` (0x3ED) · entries 2 – 7 have start block `FF FF FF FF` (-1) |

---

## 11. The PDF: who writes what

**Use the professor's NEW write-up template.** It is very different from old reports. Get it from the class page.

| Part | Writer | What to say |
|---|---|---|
| P0 | Quynh | team name, 4 names + IDs, GitHub name, repo link |
| P1 | Jake, Mike, Trieu | your dump from section 10, with 2 – 3 lines of explanation |
| P2 | Jake | the 6 VCB fields (from the lecture slide), the signature, why the VCB is written last |
| P3 | Mike | the FAT is the "FAT variation" of linked allocation (Ch. 14); the free list; why we save the free count (so we never have to count); how allocate works |
| P4 | Trieu | the 64-byte entry, 8 per block, "." and "..", a free entry = empty name + start block -1 |
| P5 | Quynh | the table below |
| P6 | Quynh (everyone checks it) | what really happened: when you met, how you met, how you split the work |
| P7 | everyone | one or more real issues each, and how you fixed them |

**P5 table (add the student IDs):**

| Part | Files | Main person | Reviewer |
|---|---|---|---|
| Volume Control Block | `vcb.h`, `vcb.c` | Jake | Quynh |
| Free space (FAT, allocate) | `freeSpace.h`, `freeSpace.c` | Mike | Jake |
| Root directory | `directory.h`, `directory.c` | Trieu | Mike |
| Main flow, Makefile | `fsInit.c`, `Makefile` | Quynh | Trieu |
| Hexdumps | PDF P1 | Jake, Mike, Trieu | everyone |
| Writing the PDF | PDF | Quynh | everyone |
| Submitting (GitHub link + PDF) | class page | Jake | Quynh |

---

## 12. Schedule

| Date | Jake | Mike | Trieu | Quynh |
|---|---|---|---|---|
| **Wed Oct 21** | commit `vcb.h` | commit `freeSpace.h` | commit `directory.h` | short team meeting |
| Oct 22 – 25 | `vcb.c` + test | `freeSpace.c` + test | `directory.c` + test | write `fsInit.c` (using the 3 `.h` files) |
| **Sun Oct 25** | merge (any order) | merge (any order) | merge (any order) | |
| **Mon Oct 26** | | | | merge `fsInit.c` + `Makefile`, run it twice |
| Tue Oct 27 | fix your own bugs | fix your own bugs | fix your own bugs | check the hexdumps with everyone |
| Wed Oct 28 | P1 + P2 | P1 + P3 | P1 + P4 | ask the professor about the FAT dump |
| Sat Oct 31 | read the PDF, add P7 | read the PDF, add P7 | read the PDF, add P7 | put the PDF together: P0, P5, P6 |
| Mon Nov 2 | | **final check** (section 13) on a fresh copy | | send the final PDF to Jake |
| **Tue Nov 3** | **submit** the GitHub link + PDF | | | |

---

## 13. Final checklist (Mike, Mon Nov 2)

- ☐ A fresh copy of the repo builds with `make`. No errors.
- ☐ The 1st `make run` says "Formatting now". The 2nd says "already formatted".
- ☐ The hexdumps match section 10 (but not the time stamps).
- ☐ Every file has the class header with the name, ID and GitHub name of **the person who wrote it**, plus the team name.
- ☐ GitHub shows commits from all four of us.
- ☐ The PDF uses the **new template** and has P0 to P7. The numbers match the dumps: 19,531 · 153 · 154 · 155 · 19,376.
- ☐ All four of us have read the PDF.
- ☐ Jake has the final PDF and the GitHub link, ready for **Tue Nov 3**.
