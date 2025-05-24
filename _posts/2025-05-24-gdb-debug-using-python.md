---
layout: post
title: "Record the program with RR and Debug with the python extension"
description: "Record the program with RR and Debug with the python extension"
date: 2025-05-24
modified: 2025-05-24
tags: [RR, GDB, Python]
---

# Using RR to Precisely Locate Linked List Corruption

When debugging complex programs, linked list corruption is a common and tricky problem. When you discover that a linked list has been corrupted, but don't know exactly when or where it happened, traditional debugging methods often fall short.This is especially challenging in production environments where the corruption might occur long before any symptoms appear. Traditional debugging approaches like adding print statements or using breakpoints are ineffective because they require you to know where to look in advance. This article will introduce how to use RR time-travel debugging capabilities, combined with a binary search strategy, to precisely locate the exact instruction that corrupted the linked list.

## Problem Background

LuaJIT's GC objects are linked in a list. We discovered that a GC object that should have been on the list couldn't be found there. By the time we detected this problem, we were far from the actual memory corruption event.
In this situation, we need to use recording and replay technology to analyze such issues.
Memory corruption issues like this are particularly insidious because:

1. They often manifest far from the actual point of corruption
2. They can cause intermittent failures that are difficult to reproduce
3. The symptoms might appear in seemingly unrelated parts of the program
4. By the time an error is detected, the program state has changed significantly

In our specific case, a LuaJIT garbage collection object was mysteriously missing from its linked list. This caused crashes during garbage collection cycles, but the actual corruption happened much earlier in the execution flow.


## Recording the Target Program

using `rr record /path/to/my/program --args` to record the target program.

For example,

```shell
rr record /usr/local/openresty/nginx/sbin/nginx
```

## Replaying the Record Snapshot

Replay the process with command `rr replay` in the recording directory.

```shell
rr replay
```

After running the above command, you'll be in the GDB terminal where you can execute standard GDB commands.

The RR provides a GDB-compatible interface with additional time-travel capabilities. This means you can use familiar GDB commands along with powerful time-travel commands like:

- `reverse-continue`: Run the program backward until a breakpoint is hit
- `reverse-step`: Step backward one source line
- `reverse-next`: Step backward, skipping over function calls
- `reverse-nexti`: Step backward one machine instruction

These commands allow you to navigate backward through the program's execution history, which is crucial for tracking down the exact moment when memory corruption occurs.


## Debugging Strategy: Instruction-level Binary Search

Our methodology comprises two principal components:

1. Utilize gdb to execute the program until reaching the nearest known breakpoint to the problematic area
2. Implement a binary search algorithm that leverages the `nexti` and `reverse-nexti` commands to identify with precision the specific instruction responsible for the linked list corruption

## Implementation Steps

### 1. Count the number of lua_resume calls

To use the binary search method, we first need to count how many times lua_resume() is called. We use the following Python script to count the total number of calls

The lua_resume function is a key entry point in the LuaJIT execution flow, making it an ideal landmark for our binary search. By counting these calls, we establish reference points that help narrow down our search space efficiently.



```py
import gdb

# Register GDB command to start the counter
class LuaResumeCounter(gdb.Command):
    def __init__(self):
        super(LuaResumeCounter, self).__init__("count-lua-resume", gdb.COMMAND_USER)

    def invoke(self, arg, from_tty):
        gdb.execute("delete")

        # Go to the beiginning of the record
        gdb.execute("reverse-continue")

        # Create a new breakpoint
        bp = gdb.Breakpoint("lua_resume")

        # Make it silent
        bp.silent = True

        # Set up a GDB variable to track the count
        gdb.execute("set $lua_resume_count = 0")

        # Define what to do when the breakpoint is hit
        command_script = """
        set $lua_resume_count = $lua_resume_count + 1
        continue
        """

        # Set the commands for the breakpoint
        bp.commands = command_script

        print("lua_resume counter started, calls will be counted silently")
        # Automatically start execution
        gdb.execute("continue")

        count = int(gdb.parse_and_eval("$lua_resume_count"))
        print(f"lua_resume has been called {count} times")

# Initialize commands
LuaResumeCounter()
```

### 2. Implement the check function

After counting the number of lua_resume() calls, we need to determine whether the target node is in the linked list when the breakpoint is triggered. Therefore, we use the following function for detection.

This check function traverses the linked list of GC objects to determine if our target object is present. We'll use this as our "oracle" during the binary search to determine whether we're examining a point in time before or after the corruption occurred.



```py
def check_address_in_list(start_addr, target_addr):
    """
    Check if a target address is present in the linked list of GCobj objects.

    Args:
        start_addr: The starting address as an integer
        target_addr: The address to look for as an integer

    Returns:
        True if the target address is found in the list, False otherwise
    """
    current_addr = start_addr

    print(f"Searching for 0x{target_addr:x} in GC list starting from 0x{start_addr:x}")

    # Set a reasonable limit to prevent infinite loops
    max_iterations = 100000
    iterations = 0

    while current_addr != 0 and iterations < max_iterations:
        # Check if this is our target
        if current_addr == target_addr:
            print(f"Found target address 0x{target_addr:x} in GC list!")
            return True

        try:
            # Cast the address to a GCobj pointer
            gcobj = gdb.parse_and_eval(f"(GCobj *)0x{current_addr:x}")

            # Get the next pointer as an integer
            next_ptr = int(gcobj["gch"]["nextgc"]["gcptr64"])

            # If we've reached the end or a cycle, break
            if next_ptr == 0:
                break
                #print("End of list reached")

            # Move to next object
            current_addr = next_ptr

        except Exception as e:
            print(f"Error accessing object at 0x{current_addr:x}: {e}")
            break

        iterations += 1

    if iterations >= max_iterations:
        print(f"Reached maximum iterations ({max_iterations}), search stopped")

    #print(f"Target address 0x{target_addr:x} not found in GC list")
    return False
```

### 3. Implement the binary search algorithm

Next, we implement binary search to locate the problematic instruction:

The binary search algorithm is particularly effective for this debugging scenario because:

1. It reduces the search space by half with each iteration
2. It has logarithmic time complexity (O(log n)), making it efficient even for large execution traces
3. It systematically narrows down the exact point of corruption without requiring manual inspection of the entire execution

Our implementation leverages GDB's Python API to automate the search process:



```python
class BinarySearch(gdb.Command):
    def __init__(self):
        super(BinarySearch, self).__init__("binary-search", gdb.COMMAND_USER)

    def invoke(self, arg, from_tty):
        # Set up GDB variables to track the count and target
        total_count = int(gdb.parse_and_eval("$lua_resume_count"))
        low = 0
        high = total_count

        gdb.execute("set pagination off")
        gdb.execute("delete")
        while low < high:
            mid = (low + high) // 2
            gdb.execute("reverse-continue")
            gdb.execute("set $lua_resume_count = 0")
            gdb.execute(f"set $target_resume_count = {mid}")
            print(f"check on {mid} hits")

            # Create a new breakpoint with proper GDB commands
            bp = gdb.Breakpoint("lua_resume")

            # Define the breakpoint commands using proper GDB syntax
            commands = """
            silent
            set $lua_resume_count = $lua_resume_count + 1
            if $lua_resume_count < $target_resume_count
                continue
            else
                printf "Stopped at lua_resume call #%d\\n", $lua_resume_count
            end
            """

            # Set the commands for the breakpoint
            bp.commands = commands

            # The following addresses is just for show
            exist = check_address_in_list(0x7ffff7bccaf8, 0x12345678)
            if exist:
                low = mid + 1
            else:
                high = mid - 1

            gdb.execute("delete")
            gdb.execute("continue")

# Initialize commands
BinarySearch()
```

### Locating the Specific Instruction

We have already located the lua_resume call closest to where the problem occurs, but this is not sufficient to troubleshoot the issue. We need to pinpoint the specific machine instruction. We can use the same binary search method as above to find the exact instruction.

At this point, we need to combine the `nexti` command to achieve instruction-level precision. Since the algorithm is similar to the steps above, we will not provide the specific code here, leaving it for interested individuals to implement.

It's important to note that at this stage, we need to limit the `nexti` search range between two consecutive lua_resume calls.

To implement the instruction-level binary search:

1. Identify the two consecutive lua_resume calls that bracket the corruption point
2. Count the number of instructions between these calls (this can be done with a simple script)
3. Apply the binary search algorithm at the instruction level using `nexti` and `reverse-nexti`
4. At each step, check if the target node is in the linked list
5. Narrow down until you find the exact instruction that causes the corruption

This approach can pinpoint the exact assembly instruction responsible for the corruption, even in complex codebases with millions of instructions between relevant points.

The power of this approach is that it works regardless of the complexity of the code path between the two lua_resume calls. Even if there are function calls, loops, or conditional branches, the binary search will methodically identify the exact instruction that causes the corruption.

## Real Debugging Example

Here is an example of our analysis using RR:
1. First, we load the record file with RR.
2. Use the source command to load the Python script.
3. Execute count-lua-resume to count the number of calls.
4. Execute binary-search to find the nearest lua_resume function call where the problem occurs.

This example demonstrates how we applied our methodology to a real-world problem in a production environment. The issue had been evading traditional debugging approaches for weeks, but we were able to identify the root cause within hours using this technique.

```gdb
$ rr replay
start 1> source luajit-debug.py
lua_resume counter script loaded
Use 'count-lua-resume' command to start counting and continue execution
Use 'binary-search' command to search the nearest position
start 1> count-lua-resume
0x00007ffff730de1a in epoll_wait (epfd=9, events=0x6fc260, maxevents=512, timeout=timeout@entry=1000) at ../sysdeps/unix/sysv/linux/epoll_wait.c:30
30      in ../sysdeps/unix/sysv/linux/epoll_wait.c
Breakpoint 1 at 0x7ffff7f03630: file lj_api.c, line 1276.
lua_resume counter started, calls will be counted silently

Program received signal SIGSTOP, Stopped (signal).
Have reached end of recorded history.
0x00007ffff730de1a in epoll_wait (epfd=9, events=0x6fc260, maxevents=512, timeout=timeout@entry=1000) at ../sysdeps/unix/sysv/linux/epoll_wait.c:30
30      in ../sysdeps/unix/sysv/linux/epoll_wait.c
lua_resume has been called 31 times
binary-search
Have reached start of recorded history.
0x00007ffff730de1a in epoll_wait (epfd=9, events=0x6fc260, maxevents=512, timeout=timeout@entry=1000) at ../sysdeps/unix/sysv/linux/epoll_wait.c:30
30      in ../sysdeps/unix/sysv/linux/epoll_wait.c
check on 15 hits
Breakpoint 2 at 0x7ffff7f03630: file lj_api.c, line 1276.
Searching for 0x12345678 in GC list starting from 0x7ffff7bccaf8
...
```

After identifying the specific lua_resume call where the corruption occurs, we proceeded with the instruction-level binary search.
The instruction-level search process is similar to the search for the nearest lua_resume call described above, so we have not provided the specific implementation.


## Conclusion

RR's time-travel debugging capabilities, when integrated with an instruction-level binary search methodology, constitute a sophisticated toolset for resolving complex issues such as linked list corruption. This analytical framework is not exclusively applicable to linked lists but can be extended to diagnose corruption across diverse data structures.

By identifying with precision the exact instruction at which corruption manifests, developers can ascertain the fundamental cause of the issue rather than addressing only its symptomatic manifestations. While this debugging approach may present initial complexity, it substantially diminishes the time and resources required when confronting problems that conventional debugging methodologies cannot effectively resolve.

### Broader Applications

The methodology described in this article extends beyond linked list corruption and can be applied to various challenging debugging scenarios:

1. **Race conditions**: By recording multiple executions and comparing them, you can identify non-deterministic behavior
2. **Memory leaks**: Trace object allocations and deallocations over time to identify patterns
3. **Stack corruption**: Pinpoint exactly when stack memory is overwritten
4. **Heap corruption**: Identify buffer overflows or use-after-free issues with precision
5. **API misuse**: Detect incorrect sequences of function calls that lead to undefined behavior

### Performance Considerations

While recording does introduce some overhead, modern implementations like RR are optimized to minimize impact:

- Typical recording overhead is 2-5x normal execution speed
- Storage requirements vary based on program complexity and recording duration
- For targeted debugging sessions, the benefits far outweigh the temporary performance impact

### Best Practices

To get the most out of this debugging approach:

1. Start with a focused recording that captures just enough context around the suspected issue
2. Use application knowledge to select appropriate landmarks (like lua_resume in our example)
3. Combine binary search with other debugging techniques like memory watchpoints
4. Document your findings to help others understand similar issues in the future
5. Consider implementing automated tests that can detect similar corruptions early


---

This article aims to provide valuable insights into advanced debugging techniques. Should you require additional clarification or further debugging strategies, please submit your inquiries in the discussion section below.
