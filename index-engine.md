# Disk-Backed Index Engine — Interview Notes

Repo: `dev-aabhinav/project3` · C++17 · CMake · GoogleTest

---

## One-line pitch (30 seconds)

A single-threaded C++17 storage engine that stores data in 4KB pages on disk,
caches them in a buffer pool with pin counts and an O(1) LRU replacer, and
builds two disk-backed indexes on top: a B+ tree (point + range queries) and
extendible hashing (point queries). I benchmarked both against the STL
containers, warm and cold, and ran a sequential-flooding experiment on the
buffer pool. *(Benchmark numbers: NOT MEASURED yet.)*

---

## Architecture

```mermaid
flowchart TD
    A["Index layer<br/>B+ tree · Extendible hash"] -->|"FetchPage / NewPage / UnpinPage"| B["Buffer pool manager<br/>frames · page table · pin count · dirty bit"]
    B -->|"Pin / Unpin / Victim"| C["Replacer<br/>LRU (later LRU-2 / scan ring)"]
    B -->|"ReadPage / WritePage / AllocatePage"| D["DiskManager<br/>4KB pages, pread/pwrite"]
    D --> E[("Data file<br/>page p at offset p × 4096")]
```

**A lookup's journey (target design; built up milestone by milestone):**
The B+ tree asks the buffer pool for the root page id. The pool checks its
page table (page id → frame). On a hit it pins the frame and returns it; on a
miss it asks the replacer for a victim frame (only unpinned frames are
candidates), writes the victim back if dirty, then has the DiskManager read
the wanted page into that frame. The tree binary-searches the node, finds the
child page id, unpins the current page and repeats until it reaches a leaf.

---

## Tech stack / concept table

| Concept | Layman meaning | Why used here | Alternative I didn't pick, and why |
|---|---|---|---|
| Fixed 4KB pages | Register book where every page is the same size | Matches OS page + filesystem block; offset = id × 4096 | 8KB/16KB (Postgres/InnoDB): higher fanout but more waste per point read; 4KB is simplest |
| `pread` / `pwrite` | "Read at this exact position" in one call | No shared file cursor, no lseek race, one syscall | `fstream`: extra user-space buffering, double copy; `mmap`: OS controls eviction, no control over write-back order (bad for a DB) |
| OS page cache (no `O_DIRECT`) | The kernel quietly keeps file data in RAM | Simple, free read-ahead | `O_DIRECT`: avoids double caching, needed for honest "cold" numbers, but needs aligned buffers |
| CMake + FetchContent | Build recipe that downloads its own test library | Reproducible on a fresh cloud box or laptop | Makefile: not portable; system-only GTest: breaks on fresh machines |
| GoogleTest fixtures | Fresh table setup before each test | Each test gets its own file, no cross-test leakage | Catch2: also fine; GTest is the industry default |
| ASan + UBSan build | A watchman for memory bugs | We `memcpy` into raw 4KB buffers; catches overflow/use-after-free | Valgrind memcheck: slower (~20–50x) |
| LRU = `std::list` + `unordered_map<key, list::iterator>` | Study table with 3 slots; return the book untouched longest | O(1) get/put; the classic interview version | Single `unordered_map` + timestamps: eviction becomes O(n) scan (or O(log n) with a heap/set) |
| Stress test vs brute force | Same as CP: compare fast solution with obviously-correct slow one on random input | Catches ordering bugs hand tests miss (caught a missing `splice` at op 81) | Only hand-written tests: miss rare interleavings |

---

## Machine specs (cloud container, recorded 2026-10-05)

- `nproc`: 4
- CPU: Intel Xeon @ 2.80GHz (family 6 model 85, Cascade Lake), KVM guest, 1 thread/core
- Caches: L1d 128 KiB (4 × 32K), L2 4 MiB (4 × 1M), L3 33 MiB
- RAM: 15 GiB, no swap
- Compilers: g++ 13.3.0, clang++ 18.1.3; CMake 3.28.3; GoogleTest 1.14
- **perf: hardware counters NOT available** (`dmesg`: "no PMU driver, software
  events only"). Cache-miss numbers must come from my laptop (real) or
  cachegrind (simulated, labelled as such).

---

## Milestones

### Milestone 1: Project skeleton + DiskManager

**What I built:** a CMake project (static library `index_engine` + GoogleTest
binary `unit_tests`) and a `DiskManager` that reads/writes whole 4KB pages
to one file with `pread`/`pwrite`, allocates page ids, and recovers its page
count when the file is reopened.

**Plain-language version (say this first)**
A database must remember data after the program closes, so it lives in a file.
A file is just one huge `char` array on disk, and disk is ~1000x slower than RAM,
so we move it in fixed blocks of 4096 bytes (like sqrt-decomposition blocks).
Page p = bytes `[p*4096, p*4096 + 4095]` (page 3 = bytes 12288..16383).
`ReadPage`/`WritePage` = memcpy one block between disk and a RAM buffer.

**MEMORISE**
- `kPageSize = 4096` (`include/index_engine/config.h`)
- `page_id_t = int32_t`, `kInvalidPageId = -1` → max 2³¹ pages × 4KB = 8 TB per file
- Class `DiskManager` (namespace `index_engine`):
  `ReadPage(page_id, char* out)`, `WritePage(page_id, const char* data)`,
  `AllocatePage()`, `Sync()` (fsync), `NumPages()`, `NumReads()`, `NumWrites()`
- Offset = `static_cast<off_t>(page_id) * 4096` — cast **before** multiplying
  (int32 × 4096 overflows at page 524,288 = 2 GB)
- Open flags: `O_RDWR | O_CREAT`, mode `0644`; no `O_DIRECT`
- Reopen: `num_pages = file_size / 4096` via `fstat`
- Unwritten-but-allocated page reads back as zeros (EOF → `memset`)
- Copy constructor/assignment deleted (owns an fd → Rule of Three)
- Default build type Release; `-DENABLE_SANITIZERS=ON` gives ASan + UBSan
- 6 tests: NewFileHasNoPages, AllocateReturnsSequentialIds,
  WriteThenReadRoundTrip, UnwrittenPageReadsAsZeros, DataSurvivesReopen,
  InvalidPageIdThrows

**UNDERSTAND**
- Why fixed-size pages: devices/OS move blocks anyway; trivial offset math.
- Short reads/writes: syscalls may move fewer bytes than asked → loop; `EINTR` → retry.
- `pwrite` returning ≠ durable. Process crash: data safe in kernel cache.
  Power loss: can be lost. Only `fsync` guarantees it reached the device.
- OS page cache = second cache under our future buffer pool ("double buffering").
- Torn pages: a 4KB write is not atomic on power loss → checksums + WAL full-page
  writes (Postgres) or doublewrite buffer (InnoDB).
- Why the DiskManager doesn't cache: single responsibility; one place to count real I/O.

**Complexity**
- `ReadPage` / `WritePage`: O(1) syscalls, O(page size) bytes copied
- `AllocatePage`: O(1) (counter increment; file grows lazily on first write)
- Space: O(1) in memory; file = num_pages × 4KB

**Interview questions**
- Why 4KB? → OS page + FS block size, one page = one block. → *Follow-up:* when 8/16KB? Higher fanout for range scans/SSD, more waste per point read, worse torn writes.
- Why pread/pwrite? → explicit offset, one syscall, no shared cursor. → *Follow-up:* short reads? Loop until 4096 bytes; retry on EINTR.
- Is data on disk after WritePage? → No, OS cache; needs fsync. → *Follow-up:* why not fsync every write? Too slow; real DBs fsync the WAL at commit, flush data pages lazily.
- Why no caching here? → buffer pool's job. → *Follow-up:* double caching with OS cache? Yes; InnoDB uses O_DIRECT to avoid it.
- Crash-safe? → No: torn pages possible. → *Follow-up:* fix? Checksums + WAL full-page images / doublewrite.

**Scaling questions**
- 1M → 1B keys? → ~16 GB raw, ~25 GB with tree overhead; int32 ids fine (8 TB). RAM runs out first: OS cache can't hold the file, reads go from ~1 µs (cached) to ~100 µs (SSD). Detect via `NumReads()`, `iostat`. Fixes: bigger pool → O_DIRECT + own pool → io_uring async I/O → sharding.
- Many threads? → pread/pwrite are thread-safe, but `num_pages_++` and counters race → `std::atomic`; then device queue depth is the limit (io_uring).
- File bigger than one disk? → segment files (Postgres 1 GB segments) or partition across machines.

**Honest weaknesses + fixes**
- No free list: deleted pages never reused, file only grows → keep a free-page list (or bitmap) persisted in a header page.
- Allocated-but-never-written trailing pages are forgotten on reopen (count comes from file size) → persist `num_pages` in a header page.
- No `O_DIRECT`: "cold" pool reads may still come from OS cache → state this in every cold benchmark, or use O_DIRECT / drop caches.
- No checksums, no torn-page detection → CRC32 per page header.
- Not thread-safe (counter races).

### Milestone 2: LRU cache (textbook version)

**What I built:** a generic header-only `LRUCache<K, V>` with O(1) `Get` and
`Put`, using `std::list` (recency order) + `std::unordered_map` (key → list
iterator) + `splice`. It is the interview version; the buffer pool will NOT
use it (see weakness below and M3).

**Plain-language version (say this first)**
Study table with room for N books. Using a book moves it to the "most recent"
end. Table full and you need a new book → send back the one untouched longest.
The map finds a book instantly; the linked list remembers the order and can
move any book to the front instantly, once the map has told you where it is.

**MEMORISE**
- File: `include/index_engine/lru_cache.h` (header-only template, no .cpp)
- `std::list<std::pair<K,V>> items_` — front = most recent, back = victim
- `std::unordered_map<K, list::iterator> index_` — `reserve(capacity)` in constructor (no rehash pauses)
- `std::optional<V> Get(const K&)` — returns `nullopt` on miss (not `-1`, which could be a real value)
- `void Put(const K&, const V&)` — update+refresh if present; else evict back if full, `emplace_front`
- Hit/update: `items_.splice(items_.begin(), items_, it)` — relinks the node, no copy, no allocation, iterator stays valid
- Evict order: read `items_.back().first` → `index_.erase(key)` → THEN `items_.pop_back()` (reverse order = use-after-free)
- Key is stored inside the list node too, so eviction knows which map entry to erase
- Capacity 0 throws `std::invalid_argument`
- 8 tests incl. stress test: 200,000 random ops, capacity 16, 40 keys, seed 12345, vs O(n) vector brute force
- Mutation check: removing `splice` in `Get` → 2 tests fail (stress test at op 81)

**UNDERSTAND**
- Why two structures: map gives O(1) find but no order; list gives O(1) move/remove but O(n) find.
- Why a *doubly* linked list: removing a node from the middle needs its prev pointer.
- Why `splice` beats erase + push_front: no free/malloc, and the iterator held by the map stays valid.
- `Get` is a *write* to the structure (it reorders) → matters for threading.
- **Why `std::list` is slow (pointer chasing):** every node is a separate heap
  allocation at a random address. Walking/touching nodes means each step is a
  likely cache miss (~100 ns to RAM vs ~1 ns L1). A vector's elements sit side by
  side, so one 64-byte cache line brings several of them at once. Plus one
  malloc/free per insert/evict.
- **What a real buffer pool uses instead:** number of frames N is fixed at startup,
  so keep `prev[N]`, `next[N]` integer arrays indexed by frame id (an "intrusive"
  list with indices as pointers). No allocation ever, contiguous memory, 4–8 bytes
  per link. Built in M3.

**Complexity**
- `Get`: O(1) average (hash lookup + splice)
- `Put`: O(1) average (hash lookup/insert/erase + list ops)
- Space: O(capacity). Per entry: a list node (2 pointers + pair + malloc header)
  plus a hash node and bucket slot. Overhead per entry is several times the size
  of a small int pair — exact bytes NOT MEASURED.

**Interview questions**
- Why map + list? → map finds in O(1), list keeps order with O(1) move. → *Follow-up:* why store the key in the node? To erase the evicted key from the map.
- Why `splice`? → moves the node in O(1) without allocating; iterators stay valid. → *Follow-up:* what breaks if you used `erase` + `push_front`? The map's stored iterator dangles unless you update it; plus an extra free + malloc.
- Write LRU live (LeetCode 146). → get: find → miss? / splice → return; put: exists? update+splice : (full? erase map(back.key), pop_back) → emplace_front, map[key]=begin.
- Is it thread-safe? → No. Even `Get` mutates the list, so a reader-writer lock doesn't help; it needs an exclusive lock. → *Follow-up:* scale it? Shard by `hash(key) % S`, each shard its own lock + LRU (RocksDB's sharded LRU cache); or CLOCK, where a hit just sets a bit.
- Weakness of LRU itself? → one big sequential scan pushes out all hot items ("sequential flooding"). → *Follow-up:* fix? LRU-2 / scan-resistant ring buffer (M13).

**Scaling questions**
- 1000x more entries? → memory overhead and malloc per insert dominate; pointer chasing means each touch is a cache miss. Fix: pre-allocated arrays with index links (M3), or CLOCK (one bit per slot).
- Many threads? → one global lock, and every `Get` needs it exclusively → lock contention. Detect: throughput flat or falling as threads increase. Fixes, cheapest first: shard into S independent LRUs → CLOCK (hits only set a bit, almost no locking) → lock-free designs. What gets harder: no single global LRU order any more, only approximate LRU per shard.

**Honest weaknesses + fixes**
- Heap allocation per node + pointer chasing → index-based frame arrays (M3).
- Not thread-safe → sharding or CLOCK.
- Pure LRU is not scan-resistant → LRU-K / ring buffer (M13).

---

## Benchmarks

All numbers: **NOT MEASURED** yet.

| # | Benchmark | Result | Conditions |
|---|---|---|---|
| 1 | Range scan: B+ tree vs `std::map`, 1M keys, warm + cold | NOT MEASURED | |
| 2 | Point lookup: extendible hash vs `std::unordered_map` (+ `std::map`) | NOT MEASURED | |
| 3 | Cache misses (perf stat / cachegrind): B+ tree vs `std::map` | NOT MEASURED — locality claim is a **hypothesis** | |
| 4 | Extendible hash growth under Zipfian skew | NOT MEASURED | |
| 5 | Sequential flooding: LRU vs fix, hit rate + p99 | NOT MEASURED | |

---

## Scaling section

*(Filled in fully at M14.)* Current design: single-threaded, single file,
no WAL, goes through OS page cache.

---

## Resume lines (DRAFT — not yet backed by measurements)

- Built a C++17 disk-backed storage engine: paged storage layer (4KB pages),
  buffer pool with LRU replacement, B+ tree with split/merge. *(merge on delete
  not built yet — M7)*
- Extendible hashing with bucket splits and directory doubling; profiled growth
  under Zipfian key skew. *(NOT MEASURED)*
- Benchmarks vs STL: *(all numbers NOT MEASURED; old "3.1x / 2.4x" claims removed
  until measured)*
