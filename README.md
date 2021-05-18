# Starve Free Reader-Writer's problem

The Reader-Writer's problem deals with synchronizing multiple processes which are categorized into 2 types namely:
- **Readers -** They read data from a shared memory location
- **Writers -** They write data to the shared memory location

Before diving into the solutions we can propose for the problem, we should know the basic implementation of a semaphore. It can also be found in the file `implementation.cpp` and is explained below

## Implementation of Semaphore

### The Process Control Block
```cpp
struct PCB{
    PCB *next;
    // All the other details of a process that a PCB contains
    .
    .
    .
    .
    .
}
```
Processes are stored in the memory using structures called PCBs. They contain important information of a process like the return address, variables, etc.

### The FIFO structure - Queue
```cpp
struct Queue{

    ProcessControlBlock *front, *rear;

    Queue(){
        front = nullptr;
        rear = nullptr;
    }

    // Function that pushes a PCB at the rear end of the queue
    void push(int pID){
        ProcessControlBlock *pcb = new ProcessControlBlock();
        pcb->ID = pID;
        if(rear == nullptr){
            front = pcb;
            rear = pcb;
        }else{
            rear->next = pcb;
            rear = pcb;
        }
    }

    // Function that pops a PCB from the front end of the queue
    ProcessControlBlock *pop(){
        if(front == nullptr){
            return nullptr;
        }else{
            ProcessControlBlock *pcb = front;
            front = front->next;
            if(front == nullptr){
                rear = nullptr;
            }
            return pcb;
        }
    }

};
```
We need a data structure to store the processes that are blocked by a semaphore. This is implemented above using a linked list of PCBs. We use queue in order to achieve FCFS implementation.

Here, we have a basic implementation of a queue with 2 pointers: `front` and `rear`, `front` representing the start of the queue and `rear` representing the end of the queue. We have two functions: `push()` and `pop()`. `push()` is used to push a process at the rear end of the queue and `pop()` is used to pop a PCB from the front of the queue.

### The Semaphore
```cpp
struct Semaphore{

    int semaphore = 1; // The actual value of the semaphore
    Semaphore(){
        semaphore = 1;
    }
    Semaphore(int s){
        semaphore = s;
    }

    // A queue to store the waiting processes
    Queue *FIFO = new Queue();

    // A function to implement the 'wait' functionality of a semaphore
    void wait(int pID){
        semaphore--;

        // If all resources are busy, push the process into the waiting queue and 'block' it
        if(semaphore < 0){
            FIFO->push(pID);
            ProcessControlBlock *pcb = FIFO->rear;
            block(pcb);
        }
    }

    // A function to implement the 'signal' functionality of a semaphore
    void signal(){
        semaphore++;

        // If there are any processes waiting for execution, pop from the waiting queue and wake the process up
        if(semaphore <= 0){
            ProcessControlBlock *pcb = FIFO->pop();
            wakeup(pcb);
        }
    }

};
```
The *Semaphore* structure consists of an integer `semaphore` with a default value of 1 and a queue `FIFO`. It also contains two functions: `wait(int pID)` and `signal()`.

- `wait(int pID)` - This function checks if any resources are available. If all resources are occupied, the function pushes the PCB corresponding to the pID into the `FIFO` queue and blocks that process, thus, removing it from the *ready queue*.
- `signal()` - This function is used when a process has completed using a resource. This function checks if there are any processes in the `FIFO` queue. If there are any processes, it pops the first process and wakes it up i.e. queues it in *ready queue*.

## Solutions to Reader-Writer's Problem
The first and second reader-writer's problems propose solutions that solve the problem, but, there is a chance that the writers may starve in the solution to the first problem and the readers may starve in the solution to the second problem. There is a third reader-writer's problem that proposes a starve-free solution. In the sections below, we will go through the classic first problem solution, the classic third problem solution and a new solution that is more efficient than the third problem.

### Classical solution (Writers Starve)
This is the solution to the first reader-writer's problem. Here, there is a chance that the writers starve. The global variables (that are shared across all the readers and writers) are as shown below.

#### Global Variables
```cpp
Semaphore *mutex = new Semaphore(1);
Semaphore *rw_mutex = new Semaphore(1);
int counter = 0;
```
There are 2 mutex locks implemented using Semaphores namely `mutex` and `rw_mutex`. `mutex` ensures the mutual exclusion of readers while accessing the variable `counter` and `rw_mutex` ensures that all the writers get access to the shared memory resource exclusively. The implementation of the reader is shown below

#### Implementation: Reader
```cpp
do{
    // Entry Section
    mutex->wait(processID);
    counter++;
    if(counter == 1) rw_mutex->wait(processID);
    mutex->signal();

    /**
     * 
     * Critical Section
     * 
    */

   // Exit Section
   mutex->wait(processID);
   counter--;
   if(counter == 0) rw_mutex->signal();
   mutex->signal();

    // Remainder Section

}while(true);
```
Here, if a reader is waiting for a writer process to signal the `rw_mutex`, all the other readers are waiting on `mutex`. After the writer process signals the `mutex`, all the readers can simultaneously perform the read operations. Till all the readers are done, all the writers are paused on `rw_mutex`, thus, causing starvation.

#### Implementation: Writer
```cpp
do{
    // Entry Section
    rw_mutex->wait(processID);
    
    /**
     * 
     * Critical Section
     * 
    */

   // Exit Section
   rw_mutex->signal();

   // Remainder Section

}while(true);
```

### Commonly Used Solution (Starve Free)
In the above solution, writers were starving as there was nothing stopping the readers entering continuously and blocking the resource for the writers. In this solution, we introduce another mutex lock implemented using a semaphore `in_mutex`. The process having access to this mutex lock can enter the workflow described in the solution above and thus, have access to the resource. This implements a check to the readers that come after the writers as all the processes are pushed into the FIFO queue of the semaphore `in_mutex`. Thus, this algorithm is starve-free.

#### Global Variables
```cpp
Semaphore *mutex = new Semaphore(1);
Semaphore *rw_mutex = new Semaphore(1);
Semaphore *in_mutex = new Semaphore(1);

int counter = 0;
```
The rest of the variables are same as the first solution with an addition of `in_mutex`. Now, both the reader and the writer implementations have to enclosed within `in_mutex` to ensure mutual exclusion in the whole process and thus, making the algorithm starve-free.

#### Implementation: Reader
```cpp
do{
    
    // Entry Section
    in_mutex->wait(processID);
    mutex->wait(processID);
    counter++;
    if(counter == 1) rw_mutex->wait(processID);
    mutex->signal();
    in_mutex->signal();

    /**
     * 
     * Critical Section
     * 
    */

   // Exit Section
   mutex->wait(processID);
   counter--;
   if(counter == 0) rw_mutex->signal();
   mutex->signal();

   // Remainder Section

}while(true);
```
Initially, `wait()` function is called for `in_mutex`. If a reader is waiting for a writer process, the reader is queued in the FIFO queue of the `in_mutex` (rather than `mutex`) with the fellow writers. Thus, in_mutex acts as a medium which ensures that all the processes have the same priority irrespective of their type being reader or writer.

#### Implementation: Writer
```cpp
do{

    // Entry Section
    in_mutex->wait(processID);
    rw_mutex->wait(processID);

    /**
     * 
     * Critical Section
     * 
    */

    // Exit Section
    rw_mutex->signal();
    in_mutex->signal();

    // Remainder Section

}while(true);
```
Like the readers, the writers are also queued in the FIFO queue of `in_mutex` by calling `wait()` for `in_mutex`.

Hence, all the processes requiring access to the resources can be scheduled in a FCFS manner. In this particular solution, we have to use 2 semaphores. An optimized solution is explained below.

### Optimized Solution (Starve Free)
#### Global Variables
Global variables shared across all the processes. They are as follows
```cpp
Semaphore *in_mutex = new Semaphore(1);
Semaphore *out_mutex = new Semaphore(1);
Semaphore *write_sem = new Semaphore(0);

int readers_started = 0; // Number of readers who have already started reading
int readers_completed = 0; // Number of readers who have completed reading
// The above variables will be changed by different semaphores
bool writer_waiting = false; // Indicates if a writer is waiting
```

#### Implementation: Reader
```cpp
do{

    // Entry Section
    in_mutex->wait(processID);
    readers_started++;
    in_mutex->signal();

    /**
     * 
     * Critical Section
     * 
    */
    
    // Exit Section
    out_mutex->wait(processID);
    readers_completed++;
    if(writer_waiting && readers_started == readers_completed){
        write_sem->signal();
    }
    out_mutex->signal();

    // Remainder section

}while(true);
```

#### Implementation: Writer
```cpp
do{

    // Entry Section
    in_mutex->wait(processID);
    out_mutex->wait(processID);
    if(readers_started == readers_completed){
        out_mutex->signal();
    }else{
        writer_waiting = true;
        out_mutex->signal();
        writer_sem->wait();
        writer_waiting = false;
    }

    /**
     * 
     * Critical Section
     * 
    */

   // Exit Section
   in_mutex->signal();

   // Remainder Section

}while(true);
```

#### Logic
This solution is similar to the solution above, with an optimization in the implementation of reader. Here too, the queue of `in_mutex` serves as a common waiting queue for the readers and the writers.

##### Logic for the writers
Firstly, the writers wait on the `in_mutex` with all the readers. After acquiring the `in_mutex`, the writer goes on to wait on the `out_mutex`. Now, after acquiring the `out_mutex` (which is just a means to introduce mutual exclusion for the variable `readers_completed`), it compares the variables `readers_started` and `readers_completed`. If they are equal, it means all the readers that had started their reading have completed it. That is, no reader is executing in its critical section currently. If that is the case, the writer simply signals the `out_mutex`, thus, releasing its control over the variable `readers_completed` and continues with its critical section. Note that any other reader or writer cannot execute in their critical sections as `in_mutex` is not signalled yet. If the variables `readers_started` and `readers_completed` are not equal i.e. there is are reader processes executing their critical sections, then, the writer changes the variable `writer_waiting` to `true` to state its presence and then, signals the `out_mutex`. Also, since the resource is busy, the writer waits in `writer_sem` for all the readers to complete their execution. Once it acquires `writer_sem`, it changes the variable `writer_waiting` back to `false` and proceeds to its critical section. Once the writer completes the critical section, it signals the `in_mutex` to state that it does not need the resource anymore. Now, the process next in queue of `in_mutex` can proceed.

##### Logic for the readers
In the previous solution, we had to enclose the entry part inside two mutex locks. But here, only one lock is sufficient. Hence, the process need not be blocked twice. This saves a great deal of time as blocking a process causes a lot of additional temporal overhead. Here, initially we have the `in_mutex`. Again, all the readers and writers have to queue in this mutex to ensure equal priority. Once a reader acquires the `in_mutex` (after a writer completes its execution or after a fellow reader signals the mutex), it shows its presence by increasing the variable `readers_started` and then immediately signals the `in_mutex`. The only thing that can keep a reader waiting, in this algorithm, is the wait for `in_mutex`. Hence, the reader directly proceeds to its critical section. Note that all the readers can read at the same time as only writers have a critical section in between the `wait()` and `signal()` methods of `in_mutex`. After the reader executes its critical section, the reader has to demark that it has completed its critical section execution and does not need the resource anymore. So, it waits for the `out_mutex` and once it acquires it, it increments the variable `readers_completed`, thus, announcing its completion of resource usage. Further, it goes to check if any writer is waiting by checking the variable `writer_waiting`. If yes, it checks if any fellow readers are executing in their critical sections. If not, it signals the writer to start its execution by calling `signal()` on the semaphore `write_sem`. After this, it signals the `out_mutex` to release the variable `readers_completed` for other readers and continues to its remainder section.


## Correctness of the starve-free solutions

### Mutual Exclusion
- In the first solution (commonly used solution), the `rw_mutex` ensures mutual exclusion among all the writers and the first reader. Also, the `in_mutex` ensures mutual exclusion among all the processes. The `mutex` ensures mutual exclusion between the readers trying to access the variable `counter`.
- In the second solution (optimized), the `in_mutex` ensures mutual exclusion among all the processes. The `out_mutex` implements mutual exclusion for the variables `readers_completed` and `writer_waiting`. The `in_mutex` serves another role of ensuring the mutual exclusion for the variable `readers_started`. Furthermore, the semaphore `write_sem` ensures mutual exclusion among the readers and the waiting writer.

### Bounded Waiting
In both the methods, before entering the critical section, all the processes must pass through `in_mutex` which stores all the waiting processes in a FIFO data structure. So, for a finite number of processes, the waiting time for any process in the queue is finite or bounded.

### Progress Requirement
Both the algorithms have such a structure that the systems cannot enter a state of deadlock. As long as the time of execution is finite for all the processes in their critical sections, there will be progress as the processes will keep on executing after waiting in the queue of `in_mutex`.

## References

1. Abraham Silberschatz, Peter B. Galvin, Greg Gagne - Operating System Concepts
2. [arXiv:1309.4507](https://arxiv.org/abs/1309.4507)
