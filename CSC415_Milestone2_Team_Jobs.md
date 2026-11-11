# Milestone 2: Team Jobs and Instructions ("Directories")

**Team:** Jake, Mike, Quynh, Trieu
**Builds on:** our Milestone 1 repo (the version we submitted on Nov 4)
**Turned in?** No. But Milestone 3 needs all of this, and the final writeup has an **"Approach Milestone 2"** section.
**Lecture links:** Ch.13 *Tree-Structured Directories* (absolute and relative path names, the current directory, operations on a directory: search, create, delete, list) and Ch.14 *Directory Implementation* (a linear list of entries). The slides have no step-by-step list for this milestone like Milestone 1 had.
**The professor's ParsePath:** on Nov 10 the professor shared the pseudocode for `ParsePath` and its helpers (`findInDirectory`, `LoadDirectory`, `isDEaDir`, the `ppinfo` struct, and the globals `alreadyLoadedRootDir` and `alreadyLoadedCWD`). This plan follows it **exactly**: the same names, the same steps and the same return values. (`DE` in his code = our `DirectoryEntry`.) His pseudocode is in **section 3.3**.

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
| **ParsePath** | The professor's function that turns a path into "which directory has it, and where is it inside". Everyone uses it. |
| **ppinfo** | What ParsePath fills in: `parent`, `index` and `lastElementName`. |
| **Parent** | The directory that holds the last name. In `/docs/notes`, the parent is `docs`. |
| **Last name** | The last part of the path. In `/docs/notes`, it is `notes`. |
| **Index** | Which entry (0, 1, 2 ...) the last name is in the parent. **-1 = not there. -2 = the path is just "/".** |
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

- If the **last** name is not there, ParsePath still returns **0** ("ok"), with **index = -1**. That is what `md` wants: "the parent is fine, and the name is free".
- If a **middle** part is missing (like `/nothere/x`), ParsePath returns **-2** (not found). If a middle part is a file, it returns **-1** (invalid: you can't go into a file).

### 3.2 Our 5 design rules

1. **The root and the cwd stay loaded in memory** (the professor's globals `alreadyLoadedRootDir` and `alreadyLoadedCWD`). ParsePath starts from one of them. We also keep the cwd as **text** for `pwd`. When `md` or `rm` writes a directory, `refreshLoadedDirs` copies the new data into the loaded root or cwd if it is the same directory, so they are **never out of date**.
2. **Give ParsePath a copy, and free the parent with `freeParent`.** ParsePath changes the string (`strtok_r`), so every caller copies the path first. The parent it gives back can be the loaded root or cwd, which we must **not** free. So everyone calls `freeParent`: it frees the parent only if it is not the root or the cwd.
3. **The path `/` gives index -2** (the professor's sentinel). `getTargetEntry` turns that into the root's own "." entry. Then `cd /`, `ls /` and `fs_isDir("/")` need no special code, and `md /` says "already exists".
4. **One directory = 1 block = 8 entries** (".", ".." and 6 free), the same as the root. So `LoadDirectory` and `writeDirectory` are one `LBAread` or one `LBAwrite`. (Limit: 6 items per directory. See section 12.)
5. **After every change, save at once.** `md` and `rm` change the FAT (they allocate or release blocks). They call `saveVolume` right away to write the FAT and the VCB. If someone closes the shell with Ctrl-C, the volume is still correct.

### 3.3 The professor's ParsePath (pseudocode)

The professor shared this on Nov 10. Jake writes `ParsePath` from it line by line, and Trieu writes the 3 helpers it needs (`findInDirectory`, `LoadDirectory`, `isDEaDir`). Read it before you code: everyone calls `ParsePath`.

```c
// Parse Path Pseudo Code

// Helper Routines Needed:
//        int findInDirectory (DE * parent, char * name);
//            Looks for name in the directory (array of directory entries)
//            returns the index of which DE matches the name
//            if not found it returns a -1

//        DE * LoadDirectory (DE * entry);
//            Loads the "blob" associated with that directory entry
//            returns a memory pointer to the directory (array of DE's)

//        int isDEaDir (DE * entry);
//            returns true if the specified DE is a directory
//            returns false if the specified DE is NOT a directory

// Global Variables Used
//        DE * alreadyLoadedRootDir;
//            Is the root directory loaded into memory

//        DE * alreadyLoadedCWD;
//            Is the current working directory already loaded into memory

typedef struct ppinfo {
    DE * parent;                //Parent of the last element
    int index;                    //index to the DE of the last element 
                                //(i.e. 0 for ".", 1 for "..", etc. and -1 if not found)
                                //(-2 if the total path is just "/")
    char * lastElementName;        //Name of the last element
    } ppinfo;

int ParsePath (char * path, ppinfo * ppi)
    {
    DE * startParent;
    DE * parent;
    char * saveptr;
    char * token1;
    char * token2;

    if (path == NULL) return -1;

    if (path[0] == '/')
        startParent = alreadyLoadedRootDir;
    else
        startParent = alreadyLoadedCWD;

    parent = startParent;

    token1 = strtok_r (path, "/", &saveptr);

    if (token1 == NULL)
        {
        if (path[0] == '/')
            {
            //Special Case - only / specified.  (generally only useful to cd command)
            ppi->parent = parent;
            ppi->index = -2;        //Sentinel value
            ppi->lastElementName = NULL;
            return (0);
            }
        else
            return (-1);
        }
    
    while (1)
        {
        int idx = findInDirectory (parent, token1);        //find the directory entry index of token 1

        token2 = strtok_r (NULL, "/", &saveptr);

        if (token2 == NULL)        //We are on the last element - time to exit function;
            {
            ppi->parent = parent;
            ppi->index = idx;
            ppi->lastElementName = token1;
            return (0);
            }
        else
            {
            // We are not the last element of the path, therefore token1 must exist
            // and must be a directory
            if (idx == -1)
                return (-2);        //Not Found
            if (! isDEaDir(&(parent[idx])))
                return (-1);        //invalid (can not navigate to a file)

            //Here we know it exists and is a directory - Load it to be the new parent
            DE * tempParent = LoadDirectory (&parent[idx]);
            
            //We want to free the parent no longer being used, but not if it is the startParent
            if (parent != startParent) 
                {
                free (parent);
                }
            parent = tempParent;
            token1 = token2;
            }
        }     //End While
    }
```

---

## 4. Who does what

| Member | Files (only this person edits them) | Functions | Test with | Writeup paragraph | Reviewer |
|---|---|---|---|---|---|
| **Trieu** | `directory.h`, `directory.c` (add), `mkdir.c` (new), `tests/testDirectory.c` | 6 helpers + `fs_mkdir` | `tests/testDirectory.c`, then `md` in the shell | Directory functions | Jake |
| **Jake** | `path.h`, `path.c` (new), `tests/testPath.c` | `ParsePath`, `fs_setcwd`, `fs_getcwd`, `initPaths`, `cleanupPaths`, `freeParent`, `getTargetEntry`, `refreshLoadedDirs`, `getCwdStart` | `tests/testPath.c` | Path parsing + cwd | Mike |
| **Mike** | `dirRead.c` (new), `mfs.h` (only the `fdDir` struct), `tests/testDirRead.c` | `fs_opendir`, `fs_readdir`, `fs_closedir`, `fs_stat`, `fs_isDir`, `fs_isFile` | `tests/testDirRead.c` | Directory iteration + file/dir identification | Quynh |
| **Quynh** | `fsInit.h` (new), `fsInit.c`, `remove.c` (new), `Makefile`, `fsshell.c` (only the `_ON` lines) | `saveVolume`, `fs_rmdir`, `fs_delete`, connecting everything | `make run` | Main flow + rm | Trieu |

Each reviewer **uses** the code they review, so they read it carefully.

**Who needs whom** (keep the waiting short):

| Order | Part | Who needs it |
|---|---|---|
| 1st (Day 2) | Trieu's helpers in `directory.c` | everyone |
| 2nd (Day 2 – 3) | Jake's `ParsePath` in `path.c` | Mike, Trieu's `md`, Quynh's `rm` |
| 3rd (Day 3) | Mike's `dirRead.c`, Trieu's `mkdir.c`, Quynh's `remove.c` | the shell |
| 4th (Day 4) | Quynh: `initPaths` / `cleanupPaths` calls + Makefile + shell switches | the full test |

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
| `findInDirectory` (professor) | a directory (the parent), a name | the index of that name, or -1 |
| `LoadDirectory` (professor) | one directory **entry** | the directory in memory (the caller frees it), or NULL |
| `isDEaDir` (professor) | one entry | 1 if it is a used directory, 0 if not |
| `isUsedEntry` | one entry | 1 if used, 0 if free |
| `directoryEntryCount` | a directory | how many entries it has |
| `findFreeEntry` | a directory | the first free index, or -1 if full |

### Step 2 (Day 2): the helpers in `directory.c`
`LoadDirectory` needs the block size from the VCB, so also include `vcb.h` at the top of `directory.c`.

| Function | How it works |
|---|---|
| `findInDirectory` | Go through every entry. Skip the free ones. Compare the name. Give back the first match, or -1. |
| `LoadDirectory` | The entry must be a directory with a start block of 1 or more (else print "not a directory", give back NULL). The entry knows the **size**, so blocks = size ÷ block size, rounded up. Get that much memory and read all the blocks from the entry's start block (next to each other, the same rule as `writeDirectory`). |
| `isDEaDir` | Used, **and** bit 9 (the directory flag, 0x200) of `permission` is on. |
| `isUsedEntry` | A free entry has an **empty name**. Anything else is used. |
| `directoryEntryCount` | The size in "." divided by 64. For our directories: 512 ÷ 64 = **8**. |
| `findFreeEntry` | Start at entry **2** (0 and 1 are always "." and ".."). Give back the first free one, or -1. |

### Step 3 (Day 2): test it alone (`tests/testDirectory.c`)
Keep the Milestone 1 part. Add a new part: make a VCB **in memory only** (load it, init it with 19,531 blocks), so `LoadDirectory` knows the block size. Make a folder "a" **by hand** at block 160 inside the root (put a copy of a's "." into the root's first free entry, name it "a", write the root). Then read the root back with `LoadDirectory(&root[0])` and check the helpers. Also make a 20-entry directory at block 170 to check that `LoadDirectory` reads more than one block.

**Build with:** `tests/testDirectory.c` + `directory.c` + `vcb.c` + `fsLow.o`

| Check | You should see |
|---|---|
| Milestone 1 part | 1 block; entry 0 "." start 154, size 512; entry 1 ".." start 154, size 512; entry 2 "" start -1 |
| `directoryEntryCount` of the root | 8 |
| `findInDirectory` "a" / "zzz" | 2 / -1 |
| `findFreeEntry` of the root | 3 |
| `isDEaDir` on "a" | 1 |
| a's ".." | points to block 154 |
| a 20-entry directory | loads with 24 entries (3 blocks) |
| `LoadDirectory` of a free entry | NULL (it prints "not a directory") |

**Push the helpers now.** Jake and Mike need them for their tests.

### Step 4 (Day 3): `fs_mkdir` in `mkdir.c`
Include `mfs.h`, `vcb.h`, `freeSpace.h`, `directory.h`, `path.h` and `fsInit.h`. Do the steps in this order ("free the parent" always means `freeParent`):

| Step | What to do | If it fails |
|---|---|---|
| 1 | Copy the path, then `ParsePath` the copy. | print "md: bad path …", give back -1 |
| 2 | The index must be **-1** (the name is new; "/" is -2, so it counts as "already there"). The last name must be shorter than 36. | print "md: … already exists" (or "the name is too long"), free the parent, give back -1 |
| 3 | `findFreeEntry` in the parent. | print "md: the directory is full", free the parent, give back -1 |
| 4 | Ask `directoryBlocksNeeded` (8 entries, = 1 block), then `allocBlocks` that many. | print "md: no free space", free the parent, give back -1 |
| 5 | `createDirectory` with **the parent** and the new start block. (It builds "." and ".." and writes the new directory.) | `releaseBlocks` (give the block back), free the parent, give back -1 |
| 6 | In the parent's free entry, put a **copy of the new directory's "."**, then write the `lastElementName` over its name. Set the parent's "." modified time. `writeDirectory` the parent, then `refreshLoadedDirs` the parent. | |
| 7 | The FAT changed, so call `saveVolume`. Free the new directory and the parent. Give back the result. | |

**Why step 6 copies the new ".":** it already has the right start block, size, dates and permission. The parent's entry needs the same facts, just with the real name. (Our test uses the same trick.)
The `mode` from the shell is ignored: every directory gets 0755, like the root.

### Done when
- ☐ Your test gives the results above.
- ☐ On Day 4, `md docs`, `md docs/notes` and `md pics` work, and `md pics` again says "already exists" (section 10).

**Trieu's commits (example):** `Add Milestone 2 prototypes to directory.h` → `Add isUsedEntry, isDEaDir and findInDirectory` → `Add LoadDirectory, directoryEntryCount and findFreeEntry` → `Test the Milestone 2 helpers in testDirectory.c` → `Add fs_mkdir in mkdir.c`

---

## 7. Jake: `ParsePath`, `cd` and `pwd`

**The idea:** a path is like **directions in a building**: "go to floor docs, then room notes". A path that starts with `/` starts at the front door (the root). Anything else starts where you are standing now (the cwd). We follow the directions one step at a time. **The professor gave us `ParsePath`, so Jake writes it line by line from his pseudocode (section 3.3).**

**Files:** `path.h`, `path.c` (both new), `tests/testPath.c`

### Step 1 (Day 1): the contract in `path.h`
- Include `directory.h`.
- A constant **MAX_PATH = 4096** (the same as the shell's `DIRMAX_LEN`).
- The professor's struct **ppinfo**, which ParsePath fills in:

| Field | Meaning |
|---|---|
| `parent` | the parent of the last element (a directory in memory) |
| `index` | where the last element is in `parent`: 0 for ".", 1 for "..", **-1 = not found**, **-2 = the whole path is just "/"** |
| `lastElementName` | the name of the last element (NULL for "/") |

- Prototypes: `ParsePath` (0 = ok, -1 = invalid, -2 = not found), `initPaths`, `cleanupPaths`, `freeParent`, `getTargetEntry`, `refreshLoadedDirs`, `getCwdStart`.

### Step 2 (Day 2): `ParsePath` and the helpers in `path.c`
Keep 3 private variables: the professor's **`alreadyLoadedRootDir`** and **`alreadyLoadedCWD`**, and the cwd **text** for `pwd` (starts as "/").

**`ParsePath`**: follow the professor's pseudocode in section 3.3 exactly:

| Step | What to do |
|---|---|
| 1 | `path` is NULL → -1. |
| 2 | `startParent` = the loaded root if the path starts with `/`, else the loaded cwd. `parent` = `startParent`. |
| 3 | `token1` = the first name (`strtok_r` on "/"). If there is none: the path is just "/" → parent, index **-2**, last name NULL, give back 0. (No name and no "/" → -1.) |
| 4 | Loop: `idx` = `findInDirectory(parent, token1)`. `token2` = the next name. |
| 5 | No `token2` → `token1` is the last element: fill in parent, index = `idx`, last name = `token1`. Give back 0. |
| 6 | Else `token1` is in the middle: `idx` -1 → give back **-2** (not found). Not a directory (`isDEaDir`) → give back **-1** (invalid). |
| 7 | It is a directory → `LoadDirectory(&parent[idx])` is the new parent. Free the old parent, **but not if it is the `startParent`**. `token1` = `token2`. Go back to step 4. |

Our 2 small additions: on the error returns in step 6 we also free the parent (not the `startParent`), so nothing leaks; and if `LoadDirectory` gives back NULL, we give back -1.

"." and ".." need **no special code**: they are real entries on the disk, so `findInDirectory` finds them.

**The helpers:**

| Helper | How it works |
|---|---|
| `initPaths` | Make a made-up entry that points to the root (start block from the VCB, size = 1 block, directory flag). `LoadDirectory` it → the loaded root. The cwd = the root, text = "/". Give back 0, or -1 if the load fails. |
| `cleanupPaths` | Free the cwd (if it is not the root), then the root. |
| `freeParent` | Free `ppi->parent` **only if** it is not the loaded root and not the loaded cwd. |
| `getTargetEntry` | Index -2 → the root's own "." entry. Index -1 → NULL. Else → `&parent[index]`. |
| `refreshLoadedDirs` | After `md` or `rm` writes a directory: if it has the same start block as the loaded root (or cwd) but is a **different copy**, copy its bytes into the loaded one. |
| `getCwdStart` | The start block of the loaded cwd (its "." entry). Quynh's `rm` uses it. |

### Step 3 (Day 3): `cd` and `pwd`
**`makeCleanPath`** (private helper, for the `pwd` text): put the cwd text in front if the path is relative. Then go through the parts **like a stack**: a name is pushed, ".." pops one (at the root it stays at the root), "." is skipped. Glue the parts back with `/`. No parts left = "/". Too long = -1.
Example: cwd `/a/b` + path `../c/./` → `/a/c`.

**`fs_setcwd`:**
1. Build the new text with `makeCleanPath`. (Nothing changes yet.)
2. Copy the path, `ParsePath` the copy. Use `getTargetEntry`: it must be found and be a directory. If not, free the parent and give back -1.
3. If its start block is the root's, the new cwd = the loaded root. Else `LoadDirectory` it.
4. `freeParent` **before** you change the cwd (the parent can be the old cwd).
5. **Only now** switch: free the old cwd (never the root), set the new cwd and the text. Give back 0.

**`fs_getcwd`:** if the caller's buffer is too small (or missing), give back NULL. Else copy the cwd text into it and give back the buffer.

### Step 4 (Day 3): test it alone (`tests/testPath.c`)
Make a VCB **in memory only** (load it, init it with 19,531 blocks, set the root start to 154). Build the tree **by hand**: root at 154, "a" at 155 inside the root, "b" at 156 inside "a", and a fake file "f.txt" in the root (no directory flag). Then call `initPaths` and try the paths and `cd`. At the end, `cleanupPaths`.

**Build with:** `tests/testPath.c` + `path.c` + `directory.c` + `vcb.c` + `fsLow.o`

| ParsePath of | You should see |
|---|---|
| `/` | returns 0, parent block 154, last NULL, index **-2** |
| `a` | returns 0, parent block 154, last "a", index 2 |
| `/a/b` | returns 0, parent block 155, last "b", index 2 |
| `a/b/..` | returns 0, parent block 156, last "..", index 1 |
| `/a/new` | returns 0, parent block 155, last "new", index **-1** |
| `/x/y` | returns **-2** (not found) |
| `/a/b/c/d` | returns **-2** |
| `/f.txt/x` | returns **-1** (a file in the middle) |
| "" (empty) | returns **-1** |

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

**Jake's commits (example):** `Add path.h` → `Add initPaths, cleanupPaths and ParsePath` → `Add freeParent, getTargetEntry, refreshLoadedDirs and getCwdStart` → `Add makeCleanPath, fs_setcwd and fs_getcwd` → `Add tests/testPath.c`

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
| `fs_opendir` | Copy the path, `ParsePath` the copy, then `getTargetEntry`. It must be found and be a directory; else `freeParent` and give back NULL. `LoadDirectory` the target (your own copy), then `freeParent`. Get memory for one `fdDir` and one `fs_diriteminfo`. (If anything fails, free what you got and give back NULL.) Set the bookmark to 0, keep the directory and the item in the `fdDir`. |
| `fs_readdir` | NULL in → NULL out. While the bookmark is before the end: take that entry, **move the bookmark**, and if the entry is used: copy its name into the item, set the type (`FT_DIRECTORY` or `FT_REGFILE`), set the record length, give back the item. At the end: NULL. |
| `fs_closedir` | Free the directory, the item and the `fdDir`. Give back 0 (-1 for NULL). |

### Step 3 (Day 3): `fs_stat`, `fs_isDir`, `fs_isFile`

| Function | How it works |
|---|---|
| `fs_stat` | Copy the path, `ParsePath`, `getTargetEntry`; it must be found. Fill in: size = the entry's size; block size = from the VCB; blocks = size in **512-byte** blocks, rounded up (so 1000 bytes = 2); access time = modified time (we have no "accessed" field); modified time; create time. `freeParent`. Give back 0 (or -1). |
| `fs_isDir` | Copy, `ParsePath`, `getTargetEntry`. 1 if found **and** a directory, else 0. A bad path is 0. `freeParent`. |
| `fs_isFile` | The same, but found **and not** a directory. |

These two give back **0** for a path that is not there. Because `getTargetEntry` turns "/" into the root's ".", `fs_isDir("/")` is 1 with no special code. The shell's `rm` and `ls` use that to decide what to do.

### Step 4 (Day 3): test it alone (`tests/testDirRead.c`)
Same setup as Jake's test (VCB in memory, root 154, "a" 155, "b" 156 by hand). Also add a **fake file** to the root: a copy of the root's ".", named "notes.txt", size 1000, start block 200, permission 0644 (no directory flag). It needs Trieu's helpers and Jake's `ParsePath`, so run it after they push. (Until then, just compile `dirRead.c` alone.) At the end, call `cleanupPaths`.

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
3. In `exitFileSystem`, replace the old save lines with one call to `saveVolume`. Then `cleanupFreeSpace`, `cleanupVCB`, "System exiting" as before. (Quynh already pushed steps 1 – 3.)

`make` and `make run` still work the same after this step.

### Step 3 (Day 3): `fs_rmdir` and `fs_delete` in `remove.c`
Include `mfs.h`, `vcb.h`, `freeSpace.h`, `directory.h`, `path.h` and `fsInit.h`. Two private helpers:

| Helper | How it works |
|---|---|
| `isDirectoryEmpty` | Entries 2 to the end must all be free. (Only "." and ".." may be used.) |
| `removeEntry` | If the entry has a start block (not -1), give the blocks back with `releaseBlocks`. (An empty file has no blocks.) Clear the entry to all zeros, then set its start block to -1 (a free entry, the same as `createDirectory` makes). Set the parent's "." modified time. `writeDirectory` the parent, then `refreshLoadedDirs` the parent. Then `saveVolume`, because the FAT changed. |

**`fs_rmdir`, check in this order:**

| Check | If it fails, print |
|---|---|
| copy the path, `ParsePath` works | "rm: bad path …" |
| it is **not** "/" (index -2), and the last name is **not** "." or ".." | "rm: cannot remove …" |
| found **and** a directory | "rm: … is not a directory" |
| it is **not the cwd** (compare with `getCwdStart`) | "rm: cannot remove the current directory" |
| load it: it is **empty** | "rm: … is not empty" |

All ok → `removeEntry`. Always `freeParent` (and free the loaded directory) before you give back.

**`fs_delete`** (for files; files come in Milestone 3, but `rm` is ready): copy, `ParsePath`; it must be found (index 0 or more) and **not** a directory ("rm: … is not a file"); then `removeEntry`.

### Step 4 (Day 4): connect it all (after everyone has pushed)
Do these together, because `initPaths` and `cleanupPaths` are in Jake's file, and the build needs every new file:
1. **fsInit.c:** at the end of `initFileSystem`, just before the last "give back 0", call `initPaths`. If it fails: print "Error: could not load the root directory.", clean up the free space and the VCB, give back -1. (So we always start in the root, "/".) In `exitFileSystem`, call `cleanupPaths` right after `saveVolume`.
2. **Makefile:** add `path.o`, `dirRead.o`, `mkdir.o` and `remove.o` to the end of the `ADDOBJ` line.
3. **fsshell.c:** turn these 5 from 0 to 1: `CMDLS_ON`, `CMDMD_ON`, `CMDRM_ON`, `CMDCD_ON`, `CMDPWD_ON`. Leave the others at 0 (they need Milestone 3).
4. `make clean`, `make`, then the full test (section 10).

### Done when
- ☐ `make` has no warnings, and the command table at start shows ls, cd, md, pwd and rm **ON**.
- ☐ Run 1 and run 2 in section 10 give the results shown.

**Quynh's commits (example):** `Add fsInit.h with saveVolume` → `Add saveVolume to fsInit.c` → `Add fs_rmdir and fs_delete in remove.c` → `Add the new files to the Makefile, call initPaths and cleanupPaths` → `Turn on ls, md, rm, cd and pwd in fsshell`

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
| **Jake** | Path parsing and the cwd: the professor's ParsePath (absolute vs. relative, token1/token2, -1 and -2), the loaded root and cwd, the text for `pwd` |
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

Milestone 2 is not turned in, but Milestone 3 needs it.

| Day | Trieu | Jake | Mike | Quynh |
|---|---|---|---|---|
| **Day 1** | `directory.h` prototypes | `path.h` | `mfs.h` fdDir | `fsInit.h` |
| **Day 2** | helpers + test, **push by evening** | `ParsePath` + helpers | opendir / readdir / closedir | `saveVolume` in fsInit.c |
| **Day 3** | `fs_mkdir` | `cd` / `pwd` + testPath | stat / isDir / isFile + testDirRead | `remove.c` |
| **Day 4** | test `md` in the shell | test `cd` / `pwd` in the shell | test `ls` in the shell | `initPaths` / `cleanupPaths` + Makefile + shell switches (morning), then the full test |
| **Day 5** | fix bugs together at our meeting: run 1, run 2, `make vrun`; reviewers read the code | | | |
| **Day 6** | each person writes their writeup paragraph (section 11) | | | |

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
