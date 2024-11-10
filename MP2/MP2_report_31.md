# OS MP2
## Trace Code

### Kernel::Kernel()
File: `threads/thread.cc`
> path: main() → Kernel::Kernel()
```cpp
Kernel::Kernel(int argc, char **argv) {
    randomSlice = FALSE;
    debugUserProg = FALSE;
    execExit = FALSE;
    consoleIn = NULL;   // default is stdin
    consoleOut = NULL;  // default is stdout
#ifndef FILESYS_STUB
    formatFlag = FALSE;
#endif
    reliability = 1;  // network reliability, default is 1.0
    hostName = 0;     // machine id, also UNIX socket name
                      // 0 is the default machine id
    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "-rs") == 0) {
            ASSERT(i + 1 < argc);
            RandomInit(atoi(argv[i + 1]));  // initialize pseudo-random
                                            // number generator
            randomSlice = TRUE;
            i++;
        } else if (strcmp(argv[i], "-s") == 0) {
            debugUserProg = TRUE;
        } else if (strcmp(argv[i], "-e") == 0) {
            execfile[++execfileNum] = argv[++i];
            cout << execfile[execfileNum] << "\n";
        } else if (strcmp(argv[i], "-ee") == 0) {
            // Added by @dasbd72
            // To end the program after all the threads are done
            execExit = TRUE;
        } ...
    }
}
```
在 `main.cc` 中會呼叫 `Kernel` 的建構子，使用從 command line 讀取到的 `argc` `argv` 當參數創建 kernel。`randomSlice` 設定成 `FALSE` 表示兩次 context switch 之間的時間固定， `execExit` 設為 `TRUE` 時，執行完所有 threads 系統會自動停止運作。`for` 迴圈內會處理從 command line 拿到的各項參數。

---

### void Kernel::ExecAll()
File: `threads/kernel.cc`
> path: Kernel::ExecAll()
```cpp
void Kernel::ExecAll() {
    for (int i = 1; i <= execfileNum; i++) {
        int a = Exec(execfile[i]);
    }
    currentThread->Finish();
    // Kernel::Exec();
}
```
目前是跑在 main thread 上，這裡會把從指令讀進來的所有 files 都開好 threads 並全部塞進 ready queue，跑完後使用 `Finish()` 讓 main thread 找到下一條 thread 執行，然後結束執行 main thread。

---

### int Kernel::Exec()
File: `threads/kernel.cc`
> path: Kernel::ExecAll() → Kernel::Exec()
```cpp
int Kernel::Exec(char *name) {
    t[threadNum] = new Thread(name, threadNum);
    t[threadNum]->setIsExec();
    t[threadNum]->space = new AddrSpace();
    t[threadNum]->Fork((VoidFunctionPtr)&ForkExecute, (void *)t[threadNum]);
    threadNum++;
    return threadNum - 1;
}
```
這裡主要是在做建構 thread 的前置作業並維護 threads table `t`，會回傳 thread 的數量。

### Thread::Thread()
File: `threads/thread.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → Thread::Thread()
```cpp
Thread::Thread(char *threadName, int threadID) {
    ID = threadID;
    name = threadName;
    isExec = false;
    stackTop = NULL;
    stack = NULL;
    status = JUST_CREATED;
    for (int i = 0; i < MachineStateSize; i++) {
        machineState[i] = NULL;  // not strictly necessary, since
                                 // new thread ignores contents
                                 // of machine registers
    }
    space = NULL;
}
```
這是 Thread 的建構子，其中 `isExec` 預設為 `false` 表示還沒被執行，而 `stack` 預設為 `NULL` 表示尚未 allocate stack。隨後的 `t[threadNum]->setIsExec();` 會再將這個 thread 設為正在執行。

### AddrSpace::AddrSpace()
File: `userprog/addrspace.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → AddrSpace::AddrSpace()
```cpp
AddrSpace::AddrSpace() {
    pageTable = new TranslationEntry[NumPhysPages];
    for (int i = 0; i < NumPhysPages; i++) {
        pageTable[i].virtualPage = i;  // for now, virt page # = phys page #
        pageTable[i].physicalPage = i;
        pageTable[i].valid = TRUE;
        pageTable[i].use = FALSE;
        pageTable[i].dirty = FALSE;
        pageTable[i].readOnly = FALSE;
    }

    // zero out the entire address space
    bzero(kernel->machine->mainMemory, MemorySize);
}
```
`t[threadNum]->space = new AddrSpace();` 會開一個跟 physical memory size 一樣大的 virtual memory，並新建一個 page table，並用 bzero 將空間的值都初始化成 0。

### void Thread::Fork()
File: `threads/thread.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → Thread::Fork()
```cpp
void Thread::Fork(VoidFunctionPtr func, void *arg) {
    Interrupt *interrupt = kernel->interrupt;
    Scheduler *scheduler = kernel->scheduler;
    IntStatus oldLevel;

    DEBUG(dbgThread, "Forking thread: " << name << " f(a): "
          << (int)func << " " << arg);
    StackAllocate(func, arg);

    oldLevel = interrupt->SetLevel(IntOff);
    scheduler->ReadyToRun(this);  // ReadyToRun assumes that interrupts
                                  // are disabled!
    (void)interrupt->SetLevel(oldLevel);
}
```
`Fork()` 會呼叫 `StackAllocate()` 去建構 stack，之後會把 interrupt 關掉，並呼叫 `ReadyToRun()` 把這個 thread 放進 ready queue 等待被執行。

### void Thread::StackAllocate()
File: `threads/thread.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → Thread::Fork() → Thread::StackAllocate()
```cpp
void Thread::StackAllocate(VoidFunctionPtr func, void *arg) {
    stack = (int *)AllocBoundedArray(StackSize * sizeof(int));
#ifdef x86
    // the x86 passes the return address on the stack.  In order for SWITCH()
    // to go to ThreadRoot when we switch to this thread, the return addres
    // used in SWITCH() must be the starting address of ThreadRoot.
    stackTop = stack + StackSize - 4;  // -4 to be on the safe side!
    *(--stackTop) = (int)ThreadRoot;
    *stack = STACK_FENCEPOST;
#endif
    machineState[PCState] = (void *)ThreadRoot;
    machineState[StartupPCState] = (void *)ThreadBegin;
    machineState[InitialPCState] = (void *)func;
    machineState[InitialArgState] = (void *)arg;
    machineState[WhenDonePCState] = (void *)ThreadFinish;
}
```
這裡會呼叫 `AllocBoundedArray()` 來創建一個stack，最後將所有參數放進 `machineState[]` 對應的 register 裡面。(`Switch.S` 會用到)

### char *AllocBoundedArray()
File: `threads/thread.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → Thread::Fork() →Thread::StackAllocate() → AllocBoundedArray()
```cpp
char *AllocBoundedArray(int size) {
#ifdef NO_MPROT
    return new char[size];
#else
    int pgSize = getpagesize();
    char *ptr = new char[pgSize * 2 + size];

    mprotect(ptr, pgSize, 0);
    mprotect(ptr + pgSize + size, pgSize, 0);
    return ptr + pgSize;
#endif
}
```
若不需要記憶體保護機制的話，`AllocBoundedArray()` 會直接開一個要求大小的記憶體空間必回傳。若需要記憶體保護的話，`AllocBoundedArray()` 會再要求的 stack 兩端加上一段距離的空白記憶體，並使用 `mprotect()` 來確保那兩段空白的記憶體不能被 access。

### void Scheduler::ReadyToRun()
File: `threads/scheduler.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → Thread::Fork() → Scheduler::ReadyToRun()
```cpp
void Scheduler::ReadyToRun(Thread *thread) {
    ASSERT(kernel->interrupt->getLevel() == IntOff);
    DEBUG(dbgThread, "Putting thread on ready list: " << thread->getName());
    // cout << "Putting thread on ready list: " << thread->getName() << endl ;
    thread->setStatus(READY);
    readyList->Append(thread);
}
```
把該條 thread 狀態設定成 `READY` 並且 append 到 `readyList` 的最後面。

### void ForkExecute()
File: `threads/thread.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → ForkExecute()
```cpp
void ForkExecute(Thread *t) {
    if (!t->space->Load(t->getName())) {
        return;  // executable not found
    }

    t->space->Execute(t->getName());
}
```
`Fork()` 的第一項參數 `ForkExecute` 會把 thread 對應到的 files load 進 memory 裡面，並確保有成功 load 之後再呼叫 `t->space->Execute(t->getName());` 執行 thread。

### bool AddrSpace::Load()
File: `userprog/addrspace.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → ForkExecute() → AddrSpace::Load()
```cpp
bool AddrSpace::Load(char *fileName) {
    OpenFile *executable = kernel->fileSystem->Open(fileName);
    NoffHeader noffH;
    unsigned int size;

    if (executable == NULL) {
        cerr << "Unable to open file " << fileName << "\n";
        return FALSE;
    }
```
一開始會用 `kernel->fileSystem->Open()` 去開檔案，若無法開啟則回傳 `FALSE` 表示 load file 失敗。
```cpp
    executable->ReadAt((char *)&noffH, sizeof(noffH), 0);
    if ((noffH.noffMagic != NOFFMAGIC) &&
        (WordToHost(noffH.noffMagic) == NOFFMAGIC))
        SwapHeader(&noffH);
    ASSERT(noffH.noffMagic == NOFFMAGIC);
```
若有成功開啟檔案的話，則呼叫 `executable->ReadAt()` 去讀取開啟的 file Noff header 的部分。`noffH.noffMagic != NOFFMAGIC` 是用來檢查目前與本地的 byte order (little endian, big endian) 是否一致，不一致的話則會用 `WordToHost()` 去調整成對應的型式，並且再確認一次是否相符，接著使用 `SwapHeader()` 將 header 的 byte order 換成正確的格式，最後用 `ASSERT()` 再確保一次增加可靠度。
```cpp
#ifdef RDATA
    // how big is address space?
    size = noffH.code.size + noffH.readonlyData.size + noffH.initData.size +
           noffH.uninitData.size + UserStackSize;
    // we need to increase the size
    // to leave room for the stack
#else
    // how big is address space?
    size = noffH.code.size + noffH.initData.size + noffH.uninitData.size + UserStackSize;  // we need to increase the size
                                                                                           // to leave room for the stack
#endif
```
計算所需空間 `size` ，需要加上 `UserStackSize` 留空間給 stack，預設大小是 1024。
```cpp
    numPages = divRoundUp(size, PageSize);
    size = numPages * PageSize;
    ASSERT(numPages <= NumPhysPages);
    DEBUG(dbgAddr, "Initializing address space: " << numPages << ", " << size);
```
`numPages = divRoundUp(size, PageSize);` 計算 pages 的數量，`divRoundUp()` 是取上界，算完 pages 的數量後將 size 調整為 `PageSize` 的整數倍，並且檢查 Pages 數量是否超過 `NumPhysPages`，防止 load 到過大的程式。
```cpp
    // then, copy in the code and data segments into memory
    // Note: this code assumes that virtual address = physical address
    if (noffH.code.size > 0) {
        DEBUG(dbgAddr, "Initializing code segment.");
        DEBUG(dbgAddr, noffH.code.virtualAddr << ", " << noffH.code.size);
        executable->ReadAt(
            &(kernel->machine->mainMemory[noffH.code.virtualAddr]),
            noffH.code.size, noffH.code.inFileAddr);
    }
    if (noffH.initData.size > 0) {
        DEBUG(dbgAddr, "Initializing data segment.");
        DEBUG(dbgAddr, noffH.initData.virtualAddr << ", " << noffH.initData.size);
        executable->ReadAt(
            &(kernel->machine->mainMemory[noffH.initData.virtualAddr]),
            noffH.initData.size, noffH.initData.inFileAddr);
    }

#ifdef RDATA
    if (noffH.readonlyData.size > 0) {
        DEBUG(dbgAddr, "Initializing read only data segment.");
        DEBUG(dbgAddr, noffH.readonlyData.virtualAddr << ", " << noffH.readonlyData.size);
        executable->ReadAt(
            &(kernel->machine->mainMemory[noffH.readonlyData.virtualAddr]),
            noffH.readonlyData.size, noffH.readonlyData.inFileAddr);
    }
#endif
```
再來會用 `ReadAt()` 將 code 和 data segments load 進 memory。
```cpp
    delete executable;  // close file
    return TRUE;        // success
}
```
最後關閉 files 並且回傳 `True` 表示成功將程式 load 進 memory。

### void AddrSpace::Execute()
File: `userprog/addrspace.cc`
> path: Kernel::ExecAll() → Kernel::Exec() → ForkExecute() → AddrSpace::Execute()
```cpp
void AddrSpace::Execute(char *fileName) {
    kernel->currentThread->space = this;
    this->InitRegisters();  // set the initial register values
    this->RestoreState();   // load page table register
    kernel->machine->Run();  // jump to the user progam
    ASSERTNOTREACHED();
}
```
`kernel->currentThread->space = this;` 會確保 kernel 有讀取到對的 page table，理論上其他地方的 code 沒有問題的話，這一行可以被省略，但是 kernel 必須確保不能有任何錯誤，因此會加上這行來增加 kernel 的可靠性。接著會將原本放在 state register 的 state restore 回來，呼叫 `kernel->machine->Run()` 開始執行這個 thread。

---

### void Thread::Finish()
File: `threads/thread.cc`
> path: Kernel::ExecAll() → Thread::Finish()
```cpp
void Thread::Finish() {
    (void)kernel->interrupt->SetLevel(IntOff);
    ASSERT(this == kernel->currentThread);

    DEBUG(dbgThread, "Finishing thread: " << name);
    if (kernel->execExit && this->getIsExec()) {
        kernel->execRunningNum--;
        if (kernel->execRunningNum == 0) {
            kernel->interrupt->Halt();
        }
    }
    Sleep(TRUE);  // invokes SWITCH
    // not reached
}
```
一開始先將 interrupt 關掉，並且確保 kernel 有跑在目前的 thread 上面。 `kernel->execExit` 為 `true` 的話表示所有 threads 執行完後會把該 program 停止。當所有 program 停止時，則呼叫 `kernel->interrupt->Halt()` 讓系統停止。但若還有其他 threads 或 program 要跑，則會呼叫 `Sleep()` 凍結這個 thread 以讓出 CPU 的資源，把 `TRUE` 傳進 `Sleep()` 表示這個 thread 已經做完了。

### void Thread::Sleep()
File: `threads/thread.cc`
> path: Kernel::ExecAll() → Thread::Finish() → Thread::Sleep()
```cpp
void Thread::Sleep(bool finishing) {
    Thread *nextThread;

    ASSERT(this == kernel->currentThread);
    ASSERT(kernel->interrupt->getLevel() == IntOff);

    DEBUG(dbgThread, "Sleeping thread: " << name);
    DEBUG(dbgTraCode, "In Thread::Sleep, Sleeping thread: " << name << ", "
          << kernel->stats->totalTicks);

    status = BLOCKED;
    // cout << "debug Thread::Sleep " << name << "wait for Idle\n";
    while ((nextThread = kernel->scheduler->FindNextToRun()) == NULL) {
        kernel->interrupt->Idle();  // no one to run, wait for an interrupt
    }
    // returns when it's time for us to run
    kernel->scheduler->Run(nextThread, finishing);
}
```
當 thread 因為某些原因需要等待的話，會執行這個 function 讓出 CPU 資源。一開始的兩個 `ASSERT` 會確保要睡眠的 thread 是目前這條 thread， 並且 interrupt 是被關閉的。`status = BLOCKED;` 會將此 thread 設為不能被調用。接著使用 `while` 呼叫 `FindNextRun()` 去尋找有沒有其他 thread 可以被執行，若沒有則會呼叫 `kernel->interrupt->Idle()` 讓 CPU 進入閒置狀態，若有找到則會呼叫 `kernel->scheduler->Run()` 去執行下一個 thread。

### Thread* Scheduler::FindNextToRun()
File: `threads/thread.cc`
> path: Kernel::ExecAll() → Thread::Finish() → Thread::Sleep() → Scheduler::FindNextToRun()
```cpp
Thread* Scheduler::FindNextToRun() {
    ASSERT(kernel->interrupt->getLevel() == IntOff);

    if (readyList->IsEmpty()) {
        return NULL;
    } else {
        return readyList->RemoveFront();
    }
}
```
如果 `readyList` 裡面沒有待執行的 thread，則回傳 NULL，若有的話則會呼叫 `readyList->RemoveFront()` 回傳下一個要被執行的 thread 並且從 `readyList` 把他 pop 掉。

### void Interrupt::Idle()
File: `machine/interrupt.cc`
> path: Kernel::ExecAll() → Thread::Finish() → Thread::Sleep() → Interrupt::Idle()
```cpp
void Interrupt::Idle() {
    status = IdleMode;
    if (CheckIfDue(TRUE)) {  // check for any pending interrupts
        status = SystemMode;
        return;  // return in case there's now a runnable thread
    }
    DEBUG(dbgInt, "Machine idle.  No interrupts to do.");
    cout << "No threads ready or runnable, and no pending interrupts.\n";
    cout << "Assuming the program completed.\n";
    Halt();
}
```
若沒有 interrupt 且沒有 thread 可以被執行，則 `Idle()` 會假設程式執行完成並呼叫 `Halt()` 停止運作。

### void Scheduler::Run()
File: `threads/scheduler.cc`
> path: Kernel::ExecAll() → Thread::Finish() → Thread::Sleep() → Scheduler::Run()

這個 function 主要是在做 thread 交換執行的步驟。
```cpp
void Scheduler::Run()Thread *nextThread, bool finishing) {
    Thread *oldThread = kernel->currentThread;

    ASSERT(kernel->interrupt->getLevel() == IntOff);

    if (finishing) {  // mark that we need to delete current thread
        ASSERT(toBeDestroyed == NULL);
        toBeDestroyed = oldThread;
    }
```
一開始一樣會先確保 interrupt 被關閉了，若傳進來的 `finishing` 為 `True`，則把舊的 thread 記在 `toBeDestroyed`。
```cpp
    if (oldThread->space != NULL) {  // if this thread is a user program,
        oldThread->SaveUserState();  // save the user's CPU registers
        oldThread->space->SaveState();
    }

    oldThread->CheckOverflow();  // check if the old thread
                                 // had an undetected stack overflow
```
接著會將當前舊 thread 的 state 存到 thread 自己的 register 以便後續繼續恢復 state 時使用。`oldThread->CheckOverflow();` 會檢查舊 thread 有無 overflow (x86 會檢查 `stack` 是否指到 `STACK_FENCEPOST`)。
```cpp
    kernel->currentThread = nextThread;  // switch to the next thread
    nextThread->setStatus(RUNNING);      // nextThread is now running

    DEBUG(dbgThread, "Switching from: " << oldThread->getName() << " to: "
          << nextThread->getName());

    SWITCH(oldThread, nextThread);

    // interrupts are off when we return from switch!
    ASSERT(kernel->interrupt->getLevel() == IntOff);

    DEBUG(dbgThread, "Now in thread: " << oldThread->getName());
```
`kernel->currentThread = nextThread;` `nextThread->setStatus(RUNNING);` 會將 kernel 自己記錄的資料更新到執行新的 thread 的狀態，更新後去呼叫 `SWITCH()` 執行實際的 thread 切換。
```cpp
    CheckToBeDestroyed();

    if (oldThread->space != NULL) {     // if there is an address space
        oldThread->RestoreUserState();  // to restore, do it.
        oldThread->space->RestoreState();
    }
}
```
`CheckToBeDestroyed()` 會檢查 `toBeDestroyed` (原本的 oldThread) 是否為 `NULL`，若不是 `NULL` 的話則用 delete 釋放記憶體。最後把一開始存在要執行的 thread register 裡面的 state 寫回 machine 的 register。`oldThread->space->RestoreState();` 會把 page table 和 page 數量切換成要執行的 thread 的資料。

### SWITCH()
File: `threads/switch.S`
> path: Kernel::ExecAll() → Thread::Finish() → Thread::Sleep() → Scheduler::Run() → SWITCH()

**綠色表是正在執行 thread 1，紅色表示正在執行 thread 2**
![image](https://hackmd.io/_uploads/S1thNE5-Je.png)

呼叫 `SWITCH()` 之後，在 switch.S 裡面會根據不同的 ISA 有不同的指令，目的是實際去切換 thread。觀察指令可以發現，他最後是跳到新 thread 的 `ra`，意思是會直接跳去新的 thread 跑。流程大致如上圖。由上圖我們也可以看到為什麼 SWITCH() 之後我們需要恢復執行狀態的是 `oldThread` 而非 `nextThread`。

## Implement page table in NachOS
### Kernel
```cpp
class Kernel {
    public:
    ...
    bool phyMemUsage[NumPhysPages]; // physical memory usage
    ...
}
```
```cpp
Kernel::Kernel(int argc, char **argv) {
    for (int i = 0; i < NumPhysPages; i++) {
        phyMemUsage[i] = FALSE;
    }
    ...
}
```
為了要記錄 physical frames 的使用狀況，所以在 kernel 裡面開一個陣列來記錄，`TRUE` 代表占用，`FALSE` 代表沒被使用，在一開始 kernel 建構的時候就會全部初始化成 `FALSE`。

### AddrSpace
```cpp
bool AddrSpace::Load(char *fileName) {
    ...
    numPages = divRoundUp(size, PageSize);
    size = numPages * PageSize;

    int availableNumPhysPages = 0;

    for (int i = 0; i < NumPhysPages; i++) {
        if (!kernel->phyMemUsage[i]) {
            availableNumPhysPages++;
        }
    }

    if (numPages > availableNumPhysPages) {
        DEBUG(dbgAddr, "Not enough memory to load " << fileName << "\n");
        ExceptionHandler(MemoryLimitException);
        return FALSE;
    }
    ...
}
```
為了要更好的分配 frames 給不同的 threads，因此將 page table 的建構移至 `Load()` 裡面實作，若還能分配的 frames 數量比新的 thread 需要的 pages 數量少的話，那就需要 raise `MemoryLimitException`，表示記憶體空間不夠分配給這個 thread。
```cpp
bool AddrSpace::Load(char *fileName) {
    ...
    pageTable = new TranslationEntry[numPages];
    for (int i = 0; i < numPages; i++) {
        pageTable[i].virtualPage = i;
        pageTable[i].valid = TRUE;
        pageTable[i].use = FALSE;
        pageTable[i].dirty = FALSE;
        pageTable[i].readOnly = FALSE;
        for (int j = 0; j < NumPhysPages; j++) {
            if (!kernel->phyMemUsage[j]) {
                pageTable[i].physicalPage = j;
                kernel->phyMemUsage[j] = TRUE;
                for (int offset = 0; offset < PageSize; offset++) {
                    kernel->machine->mainMemory[j * PageSize + offset] = 0;
                }
                break;
            }
        }
    }
    kernel->machine->pageTable = pageTable;
    kernel->machine->pageTableSize = numPages;
    ...
}
```
按照 `size` 算出來的 `numPages` 開一個對應大小的 `pageTable`，用 first fit 的方法把 virtual pages 對應到 physical frames，並且將所有 bytes 的初始值設定成 0，接著更新 `machine` 的 `pageTable` 和 `numPages`。
```cpp
bool AddrSpace::Load(char *fileName) {
    ...
    unsigned int physicalAddr;
    if (noffH.code.size > 0) {
        DEBUG(dbgAddr, "Initializing code segment.");
        DEBUG(dbgAddr, noffH.code.virtualAddr << ", " << noffH.code.size);
        Translate(noffH.code.virtualAddr, &physicalAddr, 1);
        executable->ReadAt(
            &(kernel->machine->mainMemory[physicalAddr]),
            noffH.code.size, noffH.code.inFileAddr);
    }
    if (noffH.initData.size > 0) {
        DEBUG(dbgAddr, "Initializing data segment.");
        DEBUG(dbgAddr, noffH.initData.virtualAddr << ", " << noffH.initData.size);
        Translate(noffH.initData.virtualAddr, &physicalAddr, 1);
        executable->ReadAt(
            &(kernel->machine->mainMemory[physicalAddr]),
            noffH.initData.size, noffH.initData.inFileAddr);
    }

#ifdef RDATA
    if (noffH.readonlyData.size > 0) {
        DEBUG(dbgAddr, "Initializing read only data segment.");
        DEBUG(dbgAddr, noffH.readonlyData.virtualAddr << ", " << noffH.readonlyData.size);
        Translate(noffH.readonlyData.virtualAddr, &physicalAddr, 0);
        executable->ReadAt(
            &(kernel->machine->mainMemory[physicalAddr]),
            noffH.readonlyData.size, noffH.readonlyData.inFileAddr);
    }
#endif
    ...
}
```
將要被放進 data 的 virtual address translate 成 physical address 後，再 load 進實體記憶體裡面。
```cpp
AddrSpace::~AddrSpace() {
    if (pageTable) {
        for (int i = 0; i < numPages; i++) {
            kernel->phyMemUsage[pageTable[i].physicalPage] = false;
        }
    }
    delete pageTable;
}
```
在 `AddrSpace` 的解構子裡面加上還原 `kernel->phyMemUsage` 的步驟，在 thread 結束被 delete 時會將原本占用的 frames 釋放。

### Thread
```cpp!
Thread::~Thread() {
    DEBUG(dbgThread, "Deleting thread: " << name);
    ASSERT(this != kernel->currentThread);
    if (stack != NULL)
        DeallocBoundedArray((char *)stack, StackSize * sizeof(int));
    if (space != NULL) {
        delete space;
    }
}
```
在 `Thread` 的解構子裡面新增 delete space 的功能，在 `Thread` 被解構時也要解構 spcae 以釋放空間。

### Testcase
```cpp
// consoleIO_large.c
#include "syscall.h"
#define SIZE 2048

int arr[SIZE];

int main() {
    int i;
    for (i = 0; i < SIZE; i++) {
        PrintInt(i);
    }
    return 0;
}
```
經過測試，`SIZE` 是 4096 時會 MLE，而 `SIZE` 是 2048 時不會，為了測試開啟第二個 thread 不夠的狀況，因此把 `SIZE` 設定成 2048，執行測試時開啟兩份 consoleIO_large.c 測試功能是否正常。

## Problems
1. **How does NachOS allocate the memory space for a new thread (process)?**  
    Address Space Setup:
    A new AddrSpace object is created for new thread, defining the virtual memory layout.
        
    Memory Segmentation: 
    Memory is divided into segments for code, data, and stack.
    
    Page Table Initialization: 
    A simple page table maps virtual pages to physical frames, setting up translation for the process's memory accesses.
    
    Stack Allocation: 
    Each thread within a process has a fixed-size stack at the top of the address space.
    
    Program Loading: 
    The executable code and data are loaded into memory, mapped according to the page table.

2. **How does NachOS initialize the memory content of a thread (process), including loading the user binary code in the memory?**  
    NachOS reads the code and initialized data segments from the executable file and writes them into the allocated physical memory, as specified by the page table mappings.
    

3. **How does NachOS create and manage the page table?**  
    原本的方法是在 AddrSpace 的建構子就開一個和 physical frames 一樣多的 page table，我們將其修改成按照大小分配 frames 的數量。在 thread 被刪除時會呼叫 AddrSpace 的解構子，在裡面會把原本分配給那條 thread 的 frames 還原回去。

4. **How does NachOS translate addresses?**  
    NachOS uses a page table to translate virtual addresses to physical addresses, enabling processes to work within their own address spaces independently.

5. **How NachOS initializes the machine status (registers, etc) before running a thread (process)?**  
    1. Setting Up the Program Counter
    2. Initializing the Stack Pointer 
    3. Zeroing Out Other Registers
    4. Saving and Restoring Register States
    5. Setting Up the Address Space

6. **Which object in NachOS acts the role of process control block?**  
    觀察 `Thread` 這個 class 可以發現，他存放了 thread 的執行狀態、page table...等資訊，因此在 NachOS 裡面，PCB 這個功能是由 `Thread` 這個 class 來負責。

7. **When and how does a thread get added into the ReadyToRun queue of NachOS CPU scheduler?**  
    1. Thread Creation
    2. Thread Yielding
    3. Thread Wake UP
     ReadyToRun() put this thread into ready queue and set the thread's status into "READY", waiting for execution.