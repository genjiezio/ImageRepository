# proj19-process-memory-tracker

## Final Report of group 25

---

### Result analysis

##### goal 1:
This is the goal of our design document.
![image](https://github.com/genjiezio/ImageRepository/blob/main/13.png)

We have **implemented** the basic goals and **expect** goal. 

This is the effect of **goal 1**. Which gives the informations like nums of all kinds of process state, memTotal, memFree, swapTotal, swapFree, and print the information
of main process in threads group sorted by real memory size from high to low.

The Vmrss represent the real memory the process is using, the **%mem** is calculated by Vmrss/MemTotal, was used to sort the process.

And it is **Real-time refresh**, every 3 seconds refresh 1 time.
![image](https://github.com/genjiezio/ImageRepository/blob/main/14.png)


##### goal 2:
​	For part 2, we finish the detection of memory allocation and also finish the memory leak for extension.

​	The `history_*.log` record the history of memory release and detect.

​	The `result_*.log` record the memory leaking.

​	There is an example bellow.

---


### Implementation

**Goal 1** 实时统计系统中各进程及其中包含的线程的内存使用情况：

1. **step1**: use "/proc/pid/status" to read the information of process which has Pid, and processing the data. 
   - Excute the command, store the information in a **buf**, then read the buf to a line:
   
   ``
   snprintf(buf, PATH_MAX, "/proc/%d/status", num);
   ``
   
   ``
   f = fopen(buf, "r");
   ``
   
   - Matching the information in the line with what we want, and save the information, then processing the data, such as pid:
   
   ``
   if (!strncmp(line, "Pid:", 4))
		{
			Pid = strdup(&line[4]);
		}
   ``
   
   ``
   len = strlen(Pid);
    Pid[len-1] = 0;
   ``
   
   - Extract the **main process** in **process group**(in a group, processes share virtual an real memory), the feature is **Pid=Tgid**:
   
   ``
   int a = atoi(Tgid); int b = atoi(Pid);	if(a == b){...};
   ``
   
2. **step2**: Sort the main process and maintain this struct dynamicly.
   - Define the **struct** of main process:
   
   ``
   typedef struct {
    int pid;
    string name;
	  string state;
	  int vsize;
	  int vpeak;
	  int vrss;
	  int vhwm;
    }proc ;
   ``
   
   - Use a **unordered_map** to record the relation between **real memory** and process's **id in array**, can be use to find the process by real memory.
   
   ``
   string str1 = vmrss; str1.erase(0,str1.find_first_not_of(" ")); mymap.emplace(str1, count);
   ``
   
   ``
   string str = std::to_string(x); nums[j] = mymap.at(str);
   ``
   
   - Use a **Priority_queue** to maintain the largest 30 process's real memory information:
   
   ``
   priority_queue<int, vector<int>, greater<int> > q;
   ``
   
   ``
   if(qsize < 30){
			q.push(size);
			qsize++;
		}else{
			if(size > q.top()){
				q.push(size);
				q.pop();
			}
		}
   ``

3. **step3**: Use "/proc/meminfo" to get memory information of RealMem info and SwapMemory info.
   - Excute the command, store the information in a **buff**, then read the buf to a line:
   
   ``
   snprintf(buff, PATH_MAX, "/proc/meminfo", num);
   ``
   
   ``
   f = fopen(buff, "r");
   ``
   
   - Matching the information in the line, find memory information, and processing data, such as MemTotal:
   
   ``
   if (!strncmp(line, "MemTotal:", 9))
		{
			Pid = strdup(&line[9]);
		}
   ``
   
   ``
   len = strlen(MemTotal);
    MemTotal[len-1] = 0;
   ``
   
4. **step4**: Calculate and Print the process:
   - Calculate and print, using **pringf** to control retract and space:
   
   ``
   proc p = threads[nums[y]];
		printf("%-*d",7, p.pid);
		printf("%-*s",17, p.name.c_str());
		printf("%-*s",20, p.state.c_str());
		printf("%-*d",18, p.vpeak);
		printf("%-*d",13, p.vsize);
		printf("%-*d",13, p.vhwm);
		printf("%-*d",13, p.vrss);
		double a = MemTotal, b = p.vrss;
		double d = b/a *100;
		printf("%-*.2f",10, d);
   ``
   
   - Use **printf** to clean the screen, use **sleep** to refresh every 3 seconds:
   
   ``
   printf("\033[2J");	refresh();	sleep(3);
   ``
   
   - Restart the process when fopen reach the open file number limit:
   
   ``
   execv("/proc/self/exe",args);
   ``


**Goal 2** 实现检测具体进程中内存和文件句柄的分配与释放：

1.**Before work**

​	In order to detect the memory allocation and release, we need to know the basic ways of memory allocation and release.

​	For memory allocation there are 5 types:

* `new`
* `new[]`
* `malloc`

* `calloc`

* `realloc`

​    And only 3 types for memory free:

* `delete`
* `delete[]`
* `free`

​    Thus we only need to overwrite those 8 functions to detect memory allocation and release.

2. **Data structure**

​	As before work analysis, we need to record  the informations when those 8 methods  runs.

​	We plans to output one .log document to record both informations of  allocation and free.

​	Also, we choose a double linked list to record the pointer that still not free.

​	For each node, we record the 4 kinds of informations:

* pointer
* size
* file name
* file line

​    And it is packaging in to the structure `MemoryInfo`.

```c++
struct MemoryInfo{
    void* pointer;
    size_t size;
    char const* file;
    int line;
    MemoryInfo* next;
    MemoryInfo* before;
};v
```

3. **Memory allocation**

​	When memory allocation occurs, we need to record the informations of it .

​    All of the 5 methods get the informations about size and return the pointer.

​	Thus we can recording the pointer and size over write method.

​	For file name and file line, I create a new class `MemAllocate` to pass them.

```c++
class MemAllocate{
    private:
        char const* file;
        int line;
    public:
        MemAllocate(char const* _file,int _line);
        template <class T> inline T* operator*(T* p);
};
```

​	During operator`*` of `MemAllocate`, it find the suitable node and store the file and size information in it.

4. **Memory free**

​	During the return of memory free are all void, also it is no need to record the file and line, we only need  to over write and print the message to .log document.

5. **Test case**

​	For those 8 methods, we need to test for the basic data type, class without new and class with new.

​	It is correct for basic data type, but wrong for class when use the `new` and `new[]`.

​	After we check it for several times, we find that the pointer allocations returned is 8 bigger than the pointer returns by `malloc` in it,thus we can't match it correctly.

​	Instead we check in pointer is weather in the range of the node(pointer~pointer+size).

6. **Memory leak**

​	For memory leak, it only need to delete the node when memory free.

​	And at the end of process, print the rest node in double linked list to check the memory leak.


---

### Future direction
**For goal 1** :
- The **Real-time** refresh is still not the best implementation way, we will tay to use GUI in the future.
- The information still can be expand, like the excute and wait time of process, we may do this in future

**For goal 2** :
- Cause the memory allocation and free is only for data segement, we can try to detect rest segements like heap, stack and code.

---

### Summary:
- We have learned how the top is implemented
- We have learned use "/proc/..." to get the information of process
- We have learned the mechanism of new/delete/malloc/free

---

### Division of labor

* 11810420 王照伟 : goal 1
* 11810115 陈启龙 : goal 2

---
