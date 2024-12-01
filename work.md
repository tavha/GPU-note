# GPU Architecture Note
整理最近讀GPU內容的筆記

## 序曲----先從SIMD開始
* **Flynn's Taxonomy** 
Michael J. Flynn 將 Computer 分為 4 種 architecture : [SISD](https://zh.wikipedia.org/wiki/%E5%96%AE%E6%8C%87%E4%BB%A4%E6%B5%81%E5%96%AE%E6%95%B8%E6%93%9A%E6%B5%81)、[SIMD](https://zh.wikipedia.org/wiki/%E5%8D%95%E6%8C%87%E4%BB%A4%E6%B5%81%E5%A4%9A%E6%95%B0%E6%8D%AE%E6%B5%81)、[MISD](https://zh.wikipedia.org/wiki/%E5%A4%9A%E6%8C%87%E4%BB%A4%E6%B5%81%E5%96%AE%E6%95%B8%E6%93%9A%E6%B5%81)、[MIMD](https://zh.wikipedia.org/wiki/%E5%A4%9A%E6%8C%87%E4%BB%A4%E6%B5%81%E5%A4%9A%E6%95%B0%E6%8D%AE%E6%B5%81)。其中 SIMD 為 single instruction, multiple data 的架構，將一組資料分給不同處理單元並行處理，達到 data parallelism 的效果。Vector Processor 、 GPU 便是屬於這種架構。

## 二部曲----Vector Processor
### 簡介
在還沒有 GPU 之前，Super Computer 是建築在 [Seymour Cray](https://www.digitimes.com.tw/col/article.asp?id=1398) 設計的 Vector Processor ([Cray-1](https://en.wikipedia.org/wiki/Cray-1))。因為專注於向量處理，因此相對於 CPU 更適合 supercomputing
![20241130_101309 (1)](https://hackmd.io/_uploads/BJfArluQkl.jpg =50%x)
(原文書以虛擬的VMIPS ISA 介紹)

### 硬體層面討論

* 有 **Vector Data Registers** 存向量資料(ex. 一次將 N 個 M-bits 的 data 存在 register)
* 以 **Vector Instruction** 在 Processor 上執行 (ex. 一次 load/store 64個 data(element))
* All modern vector computer have **vector functional unit with multiple parallel pipelines(lanes)** that can produce two or more results per clock cycle.
    每條 **Lane** 包含:
    * one portion of the vector register file.*
    * one execution pipeline from each vector* function unit.
* 相較於 CPU 有更多 function units (如圖)
    ![Notes_241129_233723_19b](https://hackmd.io/_uploads/r1Bh-vvQ1e.jpg =60%x)
* **Vector-Length Registers (VLEN)**
    考慮以下code:
    ```c++=
    for(int i = 0; i < n; i++){
        Y[i] = a * X[i] + Y[i]
    }
    ```
    假設 vector register 能放 64 個 element，則當 n >> 64 時，我們需要一個可以將 n 切成適當大小再去處理，這時便需要 VLR 存放每次需處理資料的大小。而這種切割技術稱為 **Strip mining**。
    ```c++=
    //strip mining 演算法
    int low = 0;
    int VL = (n % MVL); // 
    for(int j = 0; j <= (n / MVL); j++){
        for(int i = low; i < (low + VL); i++){
            Y[i] = a * X[i] + Y[i];
        }
        low += VL;
        VL = MVL;
    }
    ```
* **Vector Mask Register(VMASK)**
    當我們需要考慮在for迴圈中並非所有element都需要處理時(if條件式)，考慮以下code:
    ```c++=
    for(int i = 0; i < 64; i++){
        if(X[i] != 0){
            X[i] = X[i] - Y[i];
        }
    }
    ```
    VMASKs 為每個 element 儲存 boolean 值。1 表示通過if判斷 ; 0 表示未通過，則該 element 不需 writes back。
* **Vector Stride Register(VSTR)**
    這裡要先提到一個有趣的東東，當我們需要處理的資料散落在memory各處，例如以下 code : 
    ```c++=
    for(int i = 0; i < N; i++){
        A[i] = B[i] + C[D[i]]; //
    }
    ```
    抓取 C[] matrix 的 index 需考慮 D[] matrix 的值，則 hardware 提供 **Gather/Scatter operation** 來 handle 這種稀疏矩陣，Gather 將所有散落的資料集中到 data register 中 ; Scatter 則將做完的資料存回 memory 。用 **Stride** 來形容 "distance between 2 elements of a vector"，並將 stride 存在 VSTR。
        
### 指令層面討論
* 用 **Convoy** 來表示 the set of vector instructions that could potentially execute together. 
* **Chaining** : 在 CPU 中，我們利用 Forwarding 解決 data dependancy ，在這裡稱為 Chaining(如圖)，Data forwarding from one vector functional unit to another.
    ![Screenshot_20241130_183852_Drive](https://hackmd.io/_uploads/BJ71hDdQJg.jpg =60%x)
* Chime : a timing matric to estimate the time of a convoy. 
    * 因為如果用 clock cycle time 會被 processor-specific overload(dependent on vector length) 影響，所以使用 chime 計算一個 convoy 所需時間。
    * if convoy 大小 < vector length 長度，則假設此 convoy executes in 1 chime. 

## 三部曲----GPU
在認識了GPU後，我只想說[ NVIDIA好棒棒!!!](https://www.youtube.com/watch?v=iYWzMvlj2RQ&rco=1) 但不得不說，this part is really interesting. So, let's figure out what it happens!
**GPU**, Graphic Processing Units, 源自於 graphics accelerators。 而到現在，GPU 幾乎[主宰了 super computing](https://top500.org/lists/top500/2024/11/) 的領域。基本上 GPU 的很多觀念都來自 vector processor。
現在 PC 通常都是 CPU + GPU (如圖).![Notes_241130_192745_760](https://hackmd.io/_uploads/BJwsDu_X1l.png =70%x)

### 總覽
GPU 的 Execution Model(execution model refers to how the hardware executes the code underneath) 為 **SIMT(Single Instruction, Multiple Thread)** 或者稱 **multithreaded SIMD**, it's programmed using threads, each thread executes the same code but operates a different piece of data(如圖). 執行在同一個指令上的 threads set 被 hardware dynamically grouped into a **Warp**.
![Screenshot_20241130_200813_Drive](https://hackmd.io/_uploads/rJs2lt_Q1l.jpg =50%x)

這時就要提到 NVIDIA 的可愛東東 ---- CUDA.
**CUDA**, Compute Unified Device Architecture, produces C/C++ for the system processor(host) and a C & C++ dialect for the GPU. 簡單來說，CUDA 就是一種架構來整合 CPU 與 GPU 的工作。以下介紹其中奧妙~
### 硬體架構
以下為 GPU with many cores 的架構，主要就是由 Thread Execution Manage 管理 SM 並 load/store Global memory (有關 memory 架構後面可見)。 
![Notes_241130_194532_e21](https://hackmd.io/_uploads/SJYfiuuXJl.png =70%x)
* SM : **Streaming Multiprocessor**, 也稱為 Multithread SIMD Processor(如圖，建議放大觀看)![未_命名](https://hackmd.io/_uploads/Bk2Yh3d7kx.jpg)
* **Warp** : A set of parallel CUDA Threads that execute the same instruction together in a streaming processor. It's essentially a SIMD operation formed by hardware.(如圖)
Hardware 提供 warp scheduler ，從多個可用的 warps 中選擇一個準備好執行的 warp，並將其分配給可用的計算單元。
    ![Screenshot_20241201_095935_Drive](https://hackmd.io/_uploads/Sykj7SKmJx.jpg =70%x)


### 軟體架構
GPU 的 Programming Model ( Programming Model refers to how the programmer expresses the code ) 是 **SPMD** ( Single Program, Multiple Data)
* **Kernel** : 是在 GPU 上執行的一段 code, ex: 
    ```c++=
    __global__ void add(int *a, int *b, int *c, int n) {
        int idx = blockIdx.x * blockDim.x + threadIdx.x;
        if (idx < n) {
            c[idx] = a[idx] + b[idx];
        }
    } // code by chatgpt
    ```
* **Grid** : the code that runs on a GPU that consists of a set of Thread Blocks.
    ```c++=
    dim3 blockSize(256);  // 每個 block 包含 256 個 threads
    dim3 gridSize((n + blockSize.x - 1) /     blockSize.x); // 計算啟動多少個 blocks
    add<<<gridSize, blockSize>>>(a, b, c, n);      // 啟動 kernel
    //code by chatgpt
    ```
* **Thread Block** : It's assigned to a streaming multiprocessor that executes that code by thread block scheduler.
* **SIMD Thread** : a traditional thread contains SIMD instruction, 在硬體實現成 Warp。 
![Notes_241130_235723_6c1](https://hackmd.io/_uploads/Hy88Lh_m1g.jpg =50%x)

簡單來說，
1. Kernel 定義任務要做什麼; 
2. Grid 決定要如何分配任務到 GPU;
3. Thread Blocks 在 GPU 中 concurrent 分派任務;
4. SIMD Thread 在 GPU 中 parallel 執行任務;
![螢幕擷取畫面 2024-12-01 103151](https://hackmd.io/_uploads/BJNxTrFmye.jpg)
//~~感謝 chatgpt~~


### Memory Structure
從 CUDA 的角度來討論
* **Host** : 通常指的是 CPU 。
* **Device** : GPU 。
    ![Notes_241201_100931_f79](https://hackmd.io/_uploads/rJn3SSFQJl.jpg =50%x)
    
而在 GPU 中，Memory 又分成
*  **Global Memory** : off-chip, DRAM memory, 整顆 GPU 都可以 share。
*  **Shared memory** : On-chip, SRAM memory, local to each Streaming Multiprocessor.
*  **Local memory** : DRAM memory private to each SIMD Lane. 
*  **Register** : 在每個 Lane 上有許多 registers。
![Notes_241201_104416_66d](https://hackmd.io/_uploads/ry2xCrFm1l.jpg =70%x)

## 四部曲---- GPU其他設計討論
1. 所有 GPU 的 data transfer 都為 Gather-Scatter operation
2. 當 GPU 變得越來越快，memory bandwidth between CPU & GPU 變得更重要 ( 請看 [NVLink](https://www.nvidia.com/zh-tw/data-center/nvlink/) )
(待補)

resource :100: 
* CMU : Computer Architecture Spring 2015 
* Computer Architecture : A Quantitative Approach 
* 周志遠平行程式
* Chatgpt 大大
* 網上各種資源