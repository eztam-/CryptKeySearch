<h1 align="center">🧮 CryptKeySearch</h1>
<br>
<p align="center">
  <span>A high-performance macOS engine for Bitcoin puzzle solving</span><br>
  <em>Leveraging Apples Metal framework for accelerated private-key search on Silicon GPUs</em>
</p>

---

<img src="https://raw.githubusercontent.com/eztam-/CryptKeySearch/refs/heads/main/img/screenshot.png">

CryptKeySearch runs natively on Apple Silicon GPUs, driving high-performance private-key search at full hardware capacity.
As Apple transitioned to its Silicon architecture, many existing OpenCL-based tools, such as BitCrack, were left behind.
CryptKeySearch was written from the ground up for this new terrain.
While inspired by parts of [BitCrack’s](https://github.com/brichard19/BitCrack) concepts and architecture,
the implementation is entirely from scratch and extended by additional features like support
for modern Bitcoin address formats, including SegWit.

- Report an issue or request a feature: [new Issue](https://github.com/eztam-/CryptKeySearch/issues/new).
- Support the project: BTC: 39QmdfngkM3y4KJbwrgspNRQZvwow5BFpg


## Building and Running the Application

### Building
To build the application, Xcode must be installed from the Mac App Store. After installation, open a terminal, navigate to the project directory, and run the following command:

```
xcodebuild -scheme CryptKeySearch -destination 'platform=macOS' -configuration Release -derivedDataPath ./build
```
After the build completes successfully, you can run the application using the following command:
```
cd ./build/Build/Products/Release/

./keysearch -h 
```

### Usage
#### 1. Preparing the Address Database
Before starting a private-key search, the application builds a database containing all Bitcoin addresses to be included in the search scope.

This initialization step is required only once per address list. After the database has been created, you can run any number of key searches without repeating the import process. The database needs to be rebuilt only when you want to use a different set of addresses.

The address list must be provided as a text file, with exactly one address per line (no duplications).

To manage multiple databases, use the -d <DB FILE NAME> option to specify which database file the application should use.

```
keysearch load <path_to_your_file> 
```

#### 2. Start the key search:
```
# Example 1: Run with given start key
keysearch run -s 0000000000000000000000000000000000000000000000000000000000000001

# Example 2: Run with random start key within a given range (Puzzle 71)
keysearch run -s RANDOM:400000000000000000:7fffffffffffffffff 

# For more options see:
keysearch -h
keysearch load -h
keysearch run -h
```

### Fine Tuning
To achieve optimal performance on your specific GPU, certain parameters can be fine-tuned in `Sources/Common/Properties.swift.`
Any changes to this file require recompiling the application.

The following parameters are most relevant for performance tuning:
- **GRID_SIZE**
    This defines the number of threads submitted per batch.
    If set too high, the application may consume excessive memory and potentially slowing down the entire operating system.

- **KEYS_PER_THREAD**
    This value should be set as high as possible for maximum throughput.
    However, if set too high, it can cause memory overruns. 
    
- **THREADS_PER_THREADGROUP**
    Keep this a power of 2. If choosen too high it will be clamped to a not perfect size (a message will be printed then)
    
- **BLOOM_BIT_SIZE_PER_ITEM**
    Increase this to lower the false positive rate of the bloom filter. Or decrease to reduce memory usage (helpful for large address sets).    
    
- **RING_BUFFER_SIZE**
    If the two parameters above are set too low —or if your GPU is particularly powerful— the number of batches processed per second will increase, which introduces another performance bottleneck (see the performance optimization section below).
    In those cases, you may also need to increase the ring buffer size.


Always verify that key matching still works when experimenting with this setting!

Here are some recommended settings, tested on various GPUs:

|GPU|GRID_SIZE|KEYS_PER_THREAD|THREADS_PER_THREADGROUP|RING_BUFFER_SIZE|Measured Performance|
|---|---------|---------------|-----------------------|----------------|---------------|
|M1 Pro 16 Core|1024|1024 \* 32|128|8|157 M Keys/s|
|M5 Pro 16 Core|||||315 M Keys/s|


Also be aware, that other programms like background processes consume GPU capacity at the same time or on unpredictable schedules, which might temporary slow down the performance. You can verify this in the Activity Monitor. I have observed that sometimes a reboot helps.


## Roadmap
- Improve performance
    - secp256k1 EC claculations are partly a performance bottleneck but can still be improved further.
        - fild_mul (field multiplication) and the reduction function are the main bottleneck and can be optimized further.
    - Switch to Metal 4 which should perform better since the CPU footprint for encoding and submitting work can be kept significantly smaller.
    - Improve the address file load time for very large files.
      The loading of addresses from file into the DB and Bloomfilter is very slow, when loading large files >1GB.
      Possible solutions:
        - Disk-backed key/value store (LMDB / RocksDB / LevelDB).
        - Memory-mapped sorted file + binary search.
        - In mem hash map? -> the limit is the memory.
    - If the address list is lower than 8 or 16 use a list search instead of bloom filter (thats how it's done in Bitcrack)
- Allow users to specify a custom command to be executed on key match.
- Support for uncompressed keys has been temporary removed but could be added back.  

        
## Performance Improvement Notes
- The application is ALU bound so further optimizations should target the 32 Bit arithmetic functions. 
- One problematic bottleneck is the commandBuffer work submission to the GPU and the fact, that we can control the GPU saturation only by the following parameters:
    - Properties.KEYS_PER_THREAD = 1024
    - Properties.GRID_SIZE = 1024 x 32
  On an M1 Pro GPU this is just OK but not perfect and on faster GPUs we will run into real issues here. If we increase these two parameters, then also the memory preasure will increase because of the X and Y point buffers which increase linearly.
  So we could become memory bound if we increase this values to high. On the other hand we must increase this values to keep the GPU busy and to avoid CPU overhead, since the command encoding takes some CPU time.
  Currently I have solved this issue by introducing a ring buffer which works OK with size 4 on M1 Pro. However the batches are processed even on M1 almost to fast which can be seen in the live stats (4-5 batches per second). 
  Looking at BitCracks CL code for a reference, there is just a CPU sided loop that calls the step kernel per iteration. But in Bitcrack OpenCL, there is almost no CPU overhead as we have in Metal 3 so we need to find a different solution.
  Possible solutions:
  1. Increase the ring buffer size even further.
  2. Encode several steps per CommandEncoder (this is cenceptually already atted but uncommented in the code)
  3. Swith to Metal 4 which supports more efficient command encoding (this is partly done in branch "metal4").

    
- Mixing Jacobian and Affine (“Mixed Addition”).
  this is what other high performance OpenCL based secp256k1 implementations like hashcats do as well and I have adopted this in my implementation already.<br>
    _If both points are in Jacobian form, addition is slower because both have Z ≠ 1._<br>
   _But in most scalar multiplication algorithms (like the precomputed-table method we're using), one point is fixed — e.g. (i+1)*G — and can be stored in affine form (x, y) with Z = 1._

- What should bring measurable performance improvement:
    - Replacing the current field_inv with Fermat's Little Theorem with an optimized addition chain as. e.g. done in bitcoin-core lib



### Expected Perfromance of Apple Silicon GPUs
In general, Apple Silicon GPUs deliver strong performance for many workloads, but they are relatively weak when it comes to integer ALU operations. Solving Bitcoin puzzles relies heavily on 32-bit integer arithmetic, and CryptKeySearch is therefore primarily ALU-bound. This makes overall performance closely tied to a GPU’s integer ALU throughput.
Unfortunately, there is no widely adopted GPU benchmark that directly measures this specific capability. As a rough point of comparison, the GFXBench 5 ALU2 benchmark can be used as a proxy. By combining known Bitcrack performance data with GFXBench 5 ALU2 results for an NVIDIA RTX 2080 Ti, we can estimate a realistic performance range for Apple Silicon GPUs.

A good reference for Bitcracks average performance in a pool of GPUs is: https://btcpuzzle.info/benchmark
According to that, an RTX2080 Ti can do on average 1.4 B Keys/s.

So lets take the ALU2 benchmark results from: [here](https://nanoreview.net/en/gpu-compare/geforce-rtx-2080-ti-vs-apple-m1-pro-gpu)

RTX2080 Ti --> 4775.9 FPS    
M1 Pro (16 Core) --> 598.5 FPS

That means M1 is roughly 8 times slower than the RTX2080 Ti. Since the RTX2080 Ti can do on average 1.4 B Keys/s we can devide that by 8, which results in 175 M Keys/s which are realistic on an M1 Pro.

Doing the same math for other Apple Silicon GPUs:
| GPU  | Known avg. Bitcrack Performance |GFXBench 5 ALU2|Expected Performance|Measured CryptKeySearch Performance (Puzzle 71)|
| ------------- | ------------- |--------------|--------------|--------------|
| RTX2080 Ti  | 1.4 B Keys/s |4775.9 FPS|||
| M1 Pro (16 Core)  | n/a  |598.5 FPS|175 M Keys/s|157 M Keys/s|
| M4 (10 Core)  | n/a  |432.2 FPS|127 M Keys/s|195 M Keys/s|
| M4 Pro (20 Core)  | n/a  |912.2 FPS|267 M Keys/s||
| M4 Max (40 Core)  | n/a  |1756 FPS|518 M Keys/s||
| M5 Pro (16 Core)  | n/a  |||315 M Keys/s|



## Architecture
The GPU is used for:
* Heavy elliptic curve (secp256k1) computations
* computing hashes (SHA256 + RIPEMD160)
* Iterating over a range of private keys
* Using a bloom filter to verify addresses existing in large datasets (serveral GB)

The CPU handles:
* Managing work distribution to the GPU kernels.
* Reverse calculating addresses to their public key hashes and inserting them into a local database
* Checking bloom filter results against the database to quickly discard false-positive results.


### Address Types
|Type|Address Type|Starts With|Address Format|Public Key Format|Supported|Hash Function|
|----|------------|-----------|--------------|-----------------|---------|-------------|
|Legacy|P2PKH — Pay-to-PubKey-Hash|1|Base58Check|Compresses or Uncompressed|Yes|RIPEMD160(SHA256(pubkey))|
|Legacy|P2SH — Pay-to-Script-Hash|3|Base58Check|Compresses or Uncompressed|TBD|RIPEMD160(SHA256(redeem_script))|
|SegWit|P2WPKH — Pay-to-Witness-PubKey-Hash|bc1q|Bech32|Compressed|Yes|RIPEMD160(SHA256(pubkey))|
|SegWit|P2WSH — Pay-to-Witness-Script-Hash|bc1q|Bech32|Compressed|TBD|SHA256(script)|
|SegWit|P2SH-P2WPKH — Nested SegWit (Compatibility address)|3|Base58Check|Compressed|TBD||
|Taproot|P2TR — Pay-to-Taproot|bc1p|Bech32m|TBC|No|SHA256(xonly_pubkey) → then tweak: `tweaked_pubkey = internal_pubkey + H_tapTweak(internal_pubkey|
|Wrapped SegWit|P2WPKH-P2SH|3||TBC|No|RIPEMD160(SHA256(witness_program))|

### Address Calculation from Private Key
The following diagram shows the individual stepps to calculate a bitcoin address from a private key
<img src="https://raw.githubusercontent.com/eztam-/CryptKeySearch/refs/heads/main/img/calc_by_address_types.drawio.svg">

To make the key search loop as efficient as possible, we only want the non-reversible calculations within the loop.
The reversible calculations will be reversed before inserting the addresses from the file into the bloomfilter. 
This leads us to the following application architecture:

<img src="https://raw.githubusercontent.com/eztam-/CryptKeySearch/refs/heads/main/img/architecture.drawio.svg">

The bloom filter ingestion only happens once during application start.

### Key Search Pipeline
In general, the main bottleneck are the secp256k1 EC calculations which are very compute heavy compared to the rest of the pipeline steps.
In order to make the secp256k1 EC calculations as efficient as possible we need to avoid calculating for each private key a point multiplication to get the corresponding private key.
Instead we calculate only once in the beginning a set of base points (using costy point multiplication) and also a X and Y delta.
For consecutive batches, we only need to do point additions and inversions which is significantly faster.
We also calculate several private keys per thread.



### Endians
host-endian == little-endian on Apple Silicon GPUs/CPUs<br>
... TDB





## Stepping Model — Initialization, Hashing, and Batch Progression
The application follows a two-stage GPU processing model designed for maximum throughput:
1. Initialization (init_points_* kernel)
2. Repeated stepping (step_points_* kernel)
This design produces a characteristic and intentional pattern where each stepping pass hashes the current batch of points and then advances them to the next batch.

### 1. Initialization Kernel
The initialization kernel:
- Computes all starting public key points for the grid
- Computes the constant group-step increment ΔG = totalPoints × G
- Prepares chain buffers needed for efficient batch EC addition
**Important:**
The init kernel does not perform any hashing.
After initialization:
- xPtr/yPtr contain the points for batch 0
- No HASH160 results exist yet
This ensures that initialization remains lightweight and optimized.

### 2. Stepping Kernel
Each launch of the stepping kernel performs two operations:

#### A. Hash the current batch of points
For every point:
- Generate the public key (compressed or uncompressed)
- Compute SHA-256
- Compute RIPEMD-160
- Perform Bloom filter queries
- Write results to output buffers
These hash results always correspond to the current batch, i.e., the values in xPtr/yPtr before stepping occurs.

#### B. Advance all points by ΔG
After hashing, the kernel applies the batch-add algorithm:
`P_next = P_current + ΔG`
The updated points (P_next) are written back to xPtr/yPtr, becoming the starting points for the next batch.
This creates an intentional one-batch offset:
- Hash output → Batch N
- Updated xPtr/yPtr → Batch N+

### 3. Why this model is used
This behavior mirrors the original CUDA BitCrack implementation and is chosen because it:
- Maximizes GPU arithmetic reuse
- Keeps the inner loop simple and pipeline-friendly
- Avoids expensive hashing during initialization
- Ensures each stepping pass performs a full batch of search work
Because the initialization kernel does not hash batch 0, the first stepping kernel hashes batch 0, then advances the points to batch 1.

### Summary
- Initialization sets up batch 0 but does not hash it.
- Each stepping kernel launch performs:
  1. Hash current points → produces results for batch N
  2. Add ΔG → updates points to batch N+1
Therefore after each kernel execution:
- HASH160 results correspond to batch N,
- xPtr/yPtr contain the public key points for batch N+1.
This one-batch shift is intentional and an inherent part of the BitCrack GPU algorithm design.


## Disclaimer

This software was developed **solely for educational purposes and solving Bitcoin puzzles**.  

**Important:** Any illegal use of this software is strictly prohibited. The authors **do not endorse or accept any liability** for misuse, including but not limited to unauthorized access to systems, theft of cryptocurrencies, or any other illegal activity.

By using this software, you agree to comply with all applicable laws in your jurisdiction.

Permission is granted to use, copy, modify, and distribute this software **only for legal purposes** and for educational purposes such as solving Bitcoin puzzles. Any illegal use is prohibited.

### AI-Generated Content

Parts of this application, including code and documentation, were **generated or assisted by AI tools**. All generated content was reviewed and integrated by the project author.
