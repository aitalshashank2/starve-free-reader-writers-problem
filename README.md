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

Hence, all the processes requiring access to the resources can be scheduled in a FCFS manner.

#### Correctness of the solution

##### Mutual Exclusion
The `rw_mutex` ensures mutual exclusion among all the writers and the first reader. Also, the `in_mutex` ensures mutual exclusion among all the processes. The `mutex` ensures mutual exclusion between the readers trying to access the variable `counter`.

##### Bounded Waiting
Before entering the critical section, all the processes must pass through `in_mutex` which stores all the waiting processes in a FIFO data structure. So, for a finite number of processes, the waiting time for any process in the queue is finite or bounded.

##### Progress Requirement
There is no chance of deadlock here and all the processes complete their critical sections in finite time. So, progress is guaranteed.

In this particular solution, we have to use 2 semaphores. An optimized solution is explained below.

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
All the readers can read the data simultaneously. A writer shows itself by setting the variable `writer_waiting` to `true`.

The approach of this solution is similar to the solution of the third reader-writer's problem. Here, the semaphore `in_mutex` ensures that the writer stays in the waiting queue with the readers. This is ensured by retaining the `in_mutex` till the writer process ends its cirtical stage. Thus, all the new processes are queued up in the FIFO queue of `in_mutex` irrespective of the type of the process. The FIFO nature of the queue ensures that for a finite set of processes, all the processes are executed. Also, since the queue is FCFS, there will be no problem of starvation as all the processes will get their chance as long as the execution time in critical stage is finite for all the processes. In the writer process, we notice that we have a check to see if the number of readers reading is equal to the number of readers that have completed reading. In other words, this statement checks if there are no readers in the waiting queue. If that is the case, the writer signals both the mutex locks, one before and one after the execution of its critical section and thus, transfers control to the next process in the queue of `in_mutex`. Also, note that `writer_sem` is initialized to 0. This is because we need the semaphore to synchronize across two different processes where one process calss `wait()` and another process calls `signal()`. The reader process initially increments the `readers_started`. Then, it executes its critical section and finally, increments `readers_completed`. Now, if there are no more readers, it signals the `writer_sem` so that a writer can use the resource.

Thus, the implementation is starve-free, because the processes are queued in a FIFO manner. This approach is better than the first one because it requires only one mutex for a Reader. This is significant because using another mutex requires a lot of system calls and thus, time.

## References

1. Abraham Silberschatz, Peter B. Galvin, Greg Gagne - Operating System Concepts
2. [arXiv:1309.4507](https://arxiv.org/abs/1309.4507)
