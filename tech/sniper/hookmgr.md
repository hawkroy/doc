# Hook-Manager

## 说明

Hook Manager定义了sniper模拟器中所有预发生的模拟事件，每一个事件定义了模拟器当前运行时的一种状态，每种状态的具体含义说明见下表。当每种事件发生后，会调用系统中所有模拟Object定义的callback函数，用于完成一些特定的动作

## 结构

- std::unordered_map<HookType::hook_type_t,      std::vector\<HookCallback\> > m_registry

  使用std::unordered_map进行所有callback的管理，unordered_map利用hook_type_t（所有Hook Point）进行组织

## HookPoint

### APIs

```c++
// 注册callback
void registerHook(
  	HookType::hook_type_t type, 		// hook point
  	HookCallbackFunc func, 					// hook func, always a static class func
  	UInt64 argument, 								// hook param, always class object pointer(this)
  	HookCallbackOrder order = ORDER_NOTIFY_PRE	// call order
);

// invoke callback
SInt64 callHooks(
  	HookType::hook_type_t type, 		// hook point
  	UInt64 argument, 								// actual callback parameter, not like registerHook's argument,   it->func(it->argument(this), argument)
  	bool expect_return = false			// should return value
);
```



### CallOrder

不同的callback有不同的执行order，对应不同的call point，不代表不同的CallOrder有优先级关系

- ​      ORDER_NOTIFY_PRE,       // For callbacks that want to inspect state before any actions
- ​      ORDER_ACTION,           // For callbacks that want to change simulator state based on the event
- ​      ORDER_NOTIFY_POST,      // For callbacks that want to inspect state after any actions

### HookPoint

目前，模拟器中有如下hook point

| HookPoint                  | Description                                                  |
| -------------------------- | ------------------------------------------------------------ |
| HOOK_PERIODIC              | Barrier was reached，表明当前同步点达到，这个callback用于同步不同Core间的运行速度，保证不同Core的仿真速度尽量一致 |
| HOOK_PERIODIC_INS          | Instruction-based periodic callback，当系统中所有Core的指令指令数达到阈值(这个阈值会一直累加)，则触发对应的callback |
| HOOK_SIM_START             | Simulation start，表明sniper模拟器开始simulation             |
| HOOK_SIM_END               | Simulation end，表明sniper模拟器结束simulation               |
| HOOK_ROI_BEGIN             | ROI begin， 开始进行进行ROI进行detailed仿真                  |
| HOOK_ROI_END               | ROI end，离开ROI区域，执行fast-forwarding, icount仿真        |
| HOOK_CPUFREQ_CHANGE        | CPU frequency was changed， 设置Core的变频                   |
| HOOK_MAGIC_MARKER          | Magic marker (SimMarker) in application， 在Application中执行magic code |
| HOOK_MAGIC_USER            | Magic user function (SimUser) in application， 在Application中执行magic code |
| HOOK_INSTR_COUNT           | Core has executed a preset number of instructions， 当某个Core的指令执行数超过阈值的时候 |
| HOOK_THREAD_CREATE         | Thread creation，仿真系统中创建了新的线程                    |
| HOOK_THREAD_START          | Thread start， 当系统中某个线程开始执行                      |
| HOOK_THREAD_EXIT           | Thread end，系统中某个线程退出                               |
| HOOK_THREAD_STALL          | Thread has entered stalled state， 系统中某个线程进入stall状态 |
| HOOK_THREAD_RESUME         | Thread has entered running state， 系统中某个线程恢复执行    |
| HOOK_THREAD_MIGRATE        | Thread was moved to a different core， 系统中某个线程进行了迁移到别的Core的动作 |
| HOOK_INSTRUMENT_MODE       | Simulation mode change (ex. detailed, ffwd)， 改变simulation的模式 |
| HOOK_PRE_STAT_WRITE        | Before statistics are written (update generated stats now!)， 写入统计信息之前 |
| HOOK_SYSCALL_ENTER         | Thread enters a system call， 执行某个系统调用之前           |
| HOOK_SYSCALL_EXIT          | Thread exist from system call， 执行某个系统调用之后         |
| HOOK_APPLICATION_START     | Application (re)start， 被trace的Application开始执行         |
| HOOK_APPLICATION_EXIT      | Application exit， 被trace的Application准备退出              |
| HOOK_APPLICATION_ROI_BEGIN | ROI begin, always triggers，Application执行到ROI相对应的部分，由magicServer进行处理 |
| HOOK_APPLICATION_ROI_END   | ROI end, always triggers， Application马上退出ROI部分，由magicServer进行处理 |
| HOOK_SIGUSR1               | Sniper process received SIGUSR1，sniper收到用户发送的自定义SIGUSR1信号 |

### HookPoint Detailed

<u>**HOOK_PERIODIC**</u>

- Parameter (cast from UInt64)

  SubsecondTime 	current_time

- Callback Announcer

  BarrierSyncServer::barrierRelease

- Callback Registers

  FastForwardPerformanceManager

  SamplingManager

  SchedulerDynamic

  SyscallServer

  Python-scripting

- Description

  Barrier was reached，表明当前同步点达到，这个callback用于同步不同Core间的运行速度，保证不同Core的仿真速度尽量一致

<u>**HOOK\_PERIODIC\_INS**</u>

- Parameter (cast from UInt64)

  UInt64 icount

- Callback Announcer

  Core::hookPeriodicInsCall

- Callback Registers

  Progress

  Python-scripting

- Description

  Instruction-based periodic callback，当系统中所有Core的指令指令数达到阈值(这个阈值会一直累加)，则触发对应的callback

<u>**HOOK\_SIM\_START**</u>

- Parameter (cast from UInt64)

  none

- Callback Announcer

  Simulator::start

- Callback Registers

  Python-scripting

- Description

  Simulation start，表明sniper模拟器开始simulation

<u>**HOOK\_SIM\_END**</u>

- Parameter (cast from UInt64)

  none

- Callback Announcer

  Simulator::~Simulator

- Callback Registers

  Python-scripting

- Description

  Simulation end，表明sniper模拟器结束simulation

<u>**HOOK\_ROI\_BEGIN**</u>

- Parameter (cast from UInt64)

  none

- Callback Announcer

  MagicServer::setPerformance

- Callback Registers

  Python-scripting

  RoutineTracerThread

  SmtTimer

- Description

  ROI begin， 开始进行进行ROI进行detailed仿真

<u>**HOOK\_ROI\_END**</u>

- Parameter (cast from UInt64)

  none

- Callback Announcer

  MagicServer::setPerformance

- Callback Registers

  Python-scripting

  RoutineTracerThread

  CacheCntlr::CacheCntlr

- Description

  ROI end，离开ROI区域，执行fast-forwarding, icount仿真

<u>**HOOK\_CPUFREQ\_CHANGE**</u>

- Parameter (cast from UInt64)

  UInt64 coreid

- Callback Announcer

  MagicServer::setFrequency

- Callback Registers

  Python-scripting

- Description

  CPU frequency was changed， 设置Core的变频

<u>**HOOK\_MAGIC\_MARKER**</u>

- Parameter (cast from UInt64)

  MagicServer::MagicMarkerType *

- Callback Announcer

  MagicServer::Magic_unlocked

- Callback Registers

  Python-scripting

- Description

  Magic marker (SimMarker) in application， 在Application中执行magic code

<u>**HOOK\_MAGIC\_USER**</u>

- Parameter (cast from UInt64)

  MagicServer::MagicMarkerType *

- Callback Announcer

  MagicServer::Magic_unlocked

- Callback Registers

  Python-scripting

- Description

  Magic user function (SimUser) in application， 在Application中执行magic code

<u>**HOOK\_INSTR\_COUNT**</u>

- Parameter (cast from UInt64)

  UInt64 coreid

- Callback Announcer

  Core::countInstructions

- Callback Registers

  Python-scripting

  FastForwardPerformanceManager

  SamplingManager

- Description

  Core has executed a preset number of instructions， 当某个Core的指令执行数超过阈值的时候

<u>**HOOK\_THREAD\_CREATE**</u>

- Parameter (cast from UInt64)

  HooksManager::ThreadCreate

  ```c++
  typedef struct {
    thread_id_t thread_id;
    thread_id_t creator_thread_id;
  } ThreadCreate;
  ```

- Callback Announcer

  ThreadManager::createThread_unlocked

- Callback Registers

  Python-scripting

  ThreadStatsManager

- Description

  Thread creation，仿真系统中创建了新的线程

<u>**HOOK\_THREAD\_START**</u>

- Parameter (cast from UInt64)

  HooksManager::ThreadTime

  ```c++
  typedef struct {
    thread_id_t thread_id;
    subsecond_time_t time;
  } ThreadTime;
  ```

- Callback Announcer

  ThreadManager::onThreadStart

- Callback Registers

  Python-scripting

  ThreadStatsManager

  SmtTimer

  SchedulerDynamic

- Description

  Thread start， 当系统中某个线程开始执行

<u>**HOOK\_THREAD\_EXIT**</u>

- Parameter (cast from UInt64)

  HooksManager::ThreadTime

- Callback Announcer

  ThreadManager::onThreadExit

- Callback Registers

  Python-scripting

  ThreadStatsManager

  SmtTimer

  SchedulerDynamic

  BarrierSyncServer

- Description

  Thread end，系统中某个线程退出

<u>**HOOK\_THREAD\_STALL**</u>

- Parameter (cast from UInt64)

  HooksManager::ThreadStall

  ```c++
  typedef struct {
    thread_id_t thread_id;  						// Thread stalling
    ThreadManager::stall_type_t reason; // Reason for thread stall
    subsecond_time_t time;  						// Time at which the stall occurs (if known, else SubsecondTime::MaxTime())
  } ThreadStall;
  ```

- Callback Announcer

  ThreadManager::stallThread_async

- Callback Registers

  Python-scripting

  ThreadStatsManager

  SmtTimer

  SchedulerDynamic

  BarrierSyncServer

- Description

  Thread has entered stalled state， 系统中某个线程进入stall状态

<u>**HOOK\_THREAD\_RESUME**</u>

- Parameter (cast from UInt64)

  HooksManager::ThreadResume

  ```c++
  typedef struct {
    thread_id_t thread_id;  	// Thread being woken up
    thread_id_t thread_by;  	// Thread triggering the wakeup
    subsecond_time_t time;  	// Time at which the wakeup occurs (if known, else SubsecondTime::MaxTime())
  } ThreadResume;
  ```

- Callback Announcer

  ThreadManager::resumeThread_async

- Callback Registers

  Python-scripting

  ThreadStatsManager

  SmtTimer

  SchedulerDynamic

- Description

  Thread has entered running state， 系统中某个线程恢复执行

<u>**HOOK\_THREAD\_MIGRATE**</u>

- Parameter (cast from UInt64)

  HooksManager::ThreadMigrate

  ```c++
  typedef struct {
    thread_id_t thread_id;  		// Thread being migrated
    core_id_t core_id;      		// Core the thread is now running (or INVALID_CORE_ID == -1 for unscheduled)
    subsecond_time_t time;  		// Current time
  } ThreadMigrate;
  ```

- Callback Announcer

  ThreadManager::onThreadStart

  ThreadManager::moveThread

- Callback Registers

  Python-scripting

  SmtTimer

  BarrierSyncServer

- Description

  Thread was moved to a different core， 系统中某个线程进行了迁移到别的Core的动作

<u>**HOOK\_INSTRUMENT\_MODE**</u>

- Parameter (cast from UInt64)

  UInt64 Instrument Mode

- Callback Announcer

  Simulator::setInstrumentationMode

- Callback Registers

  Python-scripting

- Description

  Simulation mode change (ex. detailed, ffwd)， 改变simulation的模式

<u>**HOOK\_PRE\_STAT\_WRITE**</u>

- Parameter (cast from UInt64)

  const char * prefix

- Callback Announcer

  StatsManager::recordStats

- Callback Registers

  Python-scripting

  ThreadStatsManager

  CheetahManager::CheetahStats

- Description

  Before statistics are written (update generated stats now!)， 写入统计信息之前

<u>**HOOK\_SYSCALL\_ENTER**</u>

- Parameter (cast from UInt64)

  SyscallMdl::HookSyscallEnter

- Callback Announcer

  SyscallMdl::runEnter

- Callback Registers

  Python-scripting

- Description

  Thread enters a system call， 执行某个系统调用之前

<u>**HOOK\_SYSCALL\_EXIT**</u>

- Parameter (cast from UInt64)

  SyscallMdl::HookSyscallExit

- Callback Announcer

  SyscallMdl::runExit

- Callback Registers

  Python-scripting

- Description

  Thread exist from system call， 执行某个系统调用之后

<u>**HOOK\_APPLICATION\_START**</u>

- Parameter (cast from UInt64)

  app_id_t

- Callback Announcer

  TraceManager::newThread

- Callback Registers

  Python-scripting

- Description

  Application (re)start， 被trace的Application开始执行

<u>**HOOK\_APPLICATION\_EXIT**</u>

- Parameter (cast from UInt64)

  app_id_t

- Callback Announcer

  TraceManager::signalDone

- Callback Registers

  Python-scripting

- Description

  Application exit， 被trace的Application准备退出

<u>**HOOK\_APPLICATION\_ROI\_BEGIN**</u>

- Parameter (cast from UInt64)

  none

- Callback Announcer

  MagicServer::Magic_unlocked

- Callback Registers

  Python-scripting

- Description

  ROI begin, always triggers，Application执行到ROI相对应的部分，由magicServer进行处理

<u>**HOOK\_APPLICATION\_ROI\_END**</u>

- Parameter (cast from UInt64)

  none

- Callback Announcer

  MagicServer::Magic_unlocked

- Callback Registers

  Python-scripting

- Description

  ROI end, always triggers， Application马上退出ROI部分，由magicServer进行处理

<u>**HOOK\_SIGUSR1**</u>

- Parameter (cast from UInt64)

  none

- Callback Announcer

  handleSigUsr1

- Callback Registers

  Python-scripting

  CircularLog

  Logmem

  RoutineTracerOndemand

- Description

  Sniper process received SIGUSR1，sniper收到用户发送的自定义SIGUSR1信号

