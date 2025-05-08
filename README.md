# OS/161 Scheduler Implementation

## Design

This implementation provides two scheduling algorithms:

### 1. First-Come First-Serve (FCFS)

The FCFS scheduler is a simple non-preemptive scheduling algorithm that executes processes in the order they arrive in the ready queue. Once a process starts executing, it continues until it completes or voluntarily yields the CPU.

**How it works:**
- Threads are added to the end of the run queue when they become ready
- The scheduler selects the thread at the head of the run queue to run next
- No reordering of the run queue is performed

**Performance characteristics:**
- **Better performance** for workloads with similar-length CPU bursts, as it provides fair allocation of CPU time
- **Worse performance** for workloads with varying CPU burst lengths, as short processes may get stuck behind long-running ones (convoy effect)
- **No overhead** for priority calculations or queue management, making it efficient for simple workloads

### 2. Multi-Level Feedback Queue (MLFQ)

The MLFQ scheduler is a preemptive scheduling algorithm that uses multiple priority queues to favor short, interactive processes while still ensuring long-running processes make progress.

**How it works:**
- Maintains multiple priority queues (4 levels in our implementation)
- New threads start at the highest priority (level 0)
- Threads that use their entire time slice are demoted to a lower priority queue
- Periodically, all threads are moved back to the highest priority queue (priority boost) to prevent starvation
- The scheduler always selects threads from the highest priority non-empty queue

**Performance characteristics:**
- **Better performance** for interactive workloads with many short-running processes
- **Better responsiveness** for user interface and I/O-bound processes
- **Worse performance** for batch processing workloads with many long-running CPU-bound processes
- **Higher overhead** due to priority calculations, multiple queues, and priority boosts

## Implementation

### Data Structures

1. **Thread Structure Extensions:**
   - `t_priority`: Current priority level of the thread (0-3, with 0 being highest)
   - `t_cpu_time`: CPU time used by the thread since last priority adjustment
   - `t_time_slice`: Time slice allocated to the thread
   - `t_need_reschedule`: Flag to indicate if the thread needs to be rescheduled

2. **Scheduler Data:**
   - `mlfq_queues`: Array of thread lists for different priority levels
   - `mlfq_lock`: Spinlock for protecting the queues
   - `mlfq_boost_counter`: Counter for priority boost
   - `current_scheduler`: Current scheduler type (FCFS or MLFQ)

### Key Functions

1. **scheduler_bootstrap()**: Initializes the scheduler data structures
2. **scheduler_addthread()**: Adds a thread to the appropriate run queue based on the current scheduler
3. **scheduler_getnext()**: Gets the next thread to run based on the current scheduler
4. **scheduler_update_priority()**: Updates thread priority based on CPU usage
5. **scheduler_priority_boost()**: Performs a priority boost by moving all threads to the highest priority queue
6. **schedule()**: Called periodically to update thread priorities and perform scheduling decisions

### Constants

- `MLFQ_LEVELS`: Number of priority levels (4)
- `MLFQ_DEFAULT_PRIORITY`: Default priority for new threads (0, highest)
- `MLFQ_TIME_SLICE`: Default time slice (4 hardclocks)
- `MLFQ_BOOST_INTERVAL`: Priority boost interval (160 hardclocks)

## Benchmark

### Test Programs

The following test programs were used to benchmark the schedulers:

#### 1. add

The `add` program performs simple arithmetic operations. It's a short-running, CPU-bound process.

**Results:**
- **Round Robin**: Consistent performance with predictable execution time
- **FCFS**: Similar performance to Round Robin for this simple program
- **MLFQ**: Slightly better performance as it prioritizes short-running processes

#### 2. matmult

The `matmult` program performs matrix multiplication, which is a CPU-intensive operation.

**Results:**
- **Round Robin**: Provides fair time slices to all processes
- **FCFS**: May cause other processes to wait if matmult runs first
- **MLFQ**: Gradually reduces priority of matmult, allowing other processes to run

#### 3. hog

The `hog` program consumes CPU resources continuously.

**Results:**
- **Round Robin**: Ensures other processes get CPU time despite hog
- **FCFS**: Poor performance as hog can monopolize the CPU
- **MLFQ**: Effectively reduces hog's priority, minimizing its impact on other processes

#### 4. farm

The `farm` program creates multiple child processes.

**Results:**
- **Round Robin**: Distributes CPU time evenly among all processes
- **FCFS**: Processes run in creation order, which may not be optimal
- **MLFQ**: Adapts to the behavior of each child process, prioritizing interactive ones

#### 5. schedpong

The `schedpong` program tests scheduler responsiveness by measuring context switch times.

**Results:**
- **Round Robin**: Consistent but not optimal context switch times
- **FCFS**: Poor responsiveness due to non-preemptive nature
- **MLFQ**: Best responsiveness due to priority-based scheduling

## Conclusion

The MLFQ scheduler provides better overall performance for a mix of workloads, particularly for interactive systems where responsiveness is important. The FCFS scheduler is simpler and has lower overhead but can lead to poor responsiveness when long-running processes are present.
