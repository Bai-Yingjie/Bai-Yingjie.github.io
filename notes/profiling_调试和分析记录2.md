- [查看PCI的MSI中断](#查看pci的msi中断)
- [一次中断执行记录和KVM虚拟机相关的trace](#一次中断执行记录和kvm虚拟机相关的trace)
  - [中断发生在core 42上](#中断发生在core-42上)
  - [core 42的正常流程](#core-42的正常流程)

# 查看PCI的MSI中断
`/proc/interrupts`里能显示中断的信息, 但不能很方便的对应到是哪个设备的中断.

比如在我的VM里, 有:
```sh
$ cat /proc/interrupts
           CPU0       CPU1       CPU2       CPU3
  2:    1020348     924296     763104     823621     GICv3  27 Level     arch_timer
  4:        116          0          0          0     GICv3  33 Level     uart-pl011
 42:          0          0          0          0     GICv3  23 Level     arm-pmu
 43:          0          0          0          0     pl061   3 Edge      ACPI:Event
 44:          1          0          0          0   ITS-MSI 16384 Edge      aerdrv, PCIe PME, pciehp
 45:          1          0          0          0   ITS-MSI 18432 Edge      aerdrv, PCIe PME, pciehp
 46:          1          0          0          0   ITS-MSI 20480 Edge      aerdrv, PCIe PME, pciehp
 47:          1          0          0          0   ITS-MSI 22528 Edge      aerdrv, PCIe PME, pciehp
 48:          1          0          0          0   ITS-MSI 24576 Edge      aerdrv, PCIe PME, pciehp
 49:          0          0          0          0   ITS-MSI 1572864 Edge      virtio1-config
 50:          0          0          0          0   ITS-MSI 1572865 Edge      virtio1-control
 51:          0          0          0          0   ITS-MSI 1572866 Edge      virtio1-event
 52:      95112          0          0          0   ITS-MSI 1572867 Edge      virtio1-request
 53:          0          0          0          0   ITS-MSI 524288 Edge      virtio0-config
 54:          0      29863      59165          0   ITS-MSI 524289 Edge      virtio0-input.0
 55:          0          0          0          1   ITS-MSI 524290 Edge      virtio0-output.0
 56:          2          0          0          0   ITS-MSI 2097152 Edge      virtio2-config
 57:     186776          0          0          0   ITS-MSI 2097153 Edge      virtio2-input.0
 58:          0          1          0          0   ITS-MSI 2097154 Edge      virtio2-output.0
 59:          0          0          0          0   ITS-MSI 1048576 Edge      xhci_hcd
 60:          0          0          0          0   ITS-MSI 1048577 Edge      xhci_hcd
 61:          0          0          0          0   ITS-MSI 1048578 Edge      xhci_hcd
 62:          0          0          0          0   ITS-MSI 1048579 Edge      xhci_hcd
 63:          0          0          0          0   ITS-MSI 1048580 Edge      xhci_hcd
IPI0:     35739      68329      56984      48197       Rescheduling interrupts
IPI1:         5          4          3          3       Function call interrupts
IPI2:         0          0          0          0       CPU stop interrupts
IPI3:         0          0          0          0       CPU stop (for crash dump) interrupts
IPI4:         0          0          0          0       Timer broadcast interrupts
IPI5:         2          0          0          0       IRQ work interrupts
IPI6:         0          0          0          0       CPU wake-up interrupts
```

有几个中断想知道对应的设备是什么
1. 查pci, 我这里想找eth1
```bash
$ lspci
01:00.0 Ethernet controller: Red Hat, Inc Virtio network device (rev 01)
03:00.0 SCSI storage controller: Red Hat, Inc Virtio SCSI (rev 01)
04:00.0 Ethernet controller: Red Hat, Inc Virtio network device (rev 01)
```
2. 看看eth1的pci
```bash
$ ethtool -i eth1
bus-info: 0000:04:00.0
```
3. 到/sys/bus下找pci号
```bash
bai@localhost /sys/bus/pci/devices/0000:04:00.0
```
4. 找到对应的中断号
```bash
$ ls msi_irqs/
56  57  58
```
5. 可以修改一下中断route到哪个CPU, 比如下面把处理中断的CPU从0改为1
```bash
bai@localhost /proc/irq/57
$ cat effective_affinity
1
bai@localhost /proc/irq/57
$ cat smp_affinity_list
0
[root@localhost 57]# echo 1 > smp_affinity_list
[root@localhost 57]# cat smp_affinity_list
1
[root@localhost 57]# cat smp_affinity
2
```

# 一次中断执行记录和KVM虚拟机相关的trace
用ftrace的function_graph抓到的一次中断过程.
```bash
#系统通过isolcpus=2-23,26-47 只保留4个核给系统(centos 7.5)使用.
HOST: 0 1 24 25
#OVS的pmd进程跑在四个核上
OVS: 12 13 14 15
#两个VM分别pin了4个CPU
VM1: 26 27 28 29
VM2: 40 41 42 43
```

## 中断发生在core 42上
```c
gic_handle_irq()
  __handle_domain_irq() // 44 us
    irq_enter()
      rcu_irq_enter()
      tick_irq_enter()
      _local_bh_enable()
    irq_find_mapping()
    generic_handle_irq()
      handle_percpu_devid_irq()
        arch_timer_handler_phys()
          hrtimer_interrupt()
        gic_eoimode1_eoi_irq()
    irq_exit()
      idle_cpu()
      tick_nohz_irq_exit()
      rcu_irq_exit()
  __handle_domain_irq() // 120 us
    irq_enter()
    irq_find_mapping()
    generic_handle_irq()
      handle_percpu_devid_irq()
        arch_timer_handler_phys()
          hrtimer_interrupt()
            __hrtimer_run_queues()
              __remove_hrtimer()
              kvm_timer_expire()
                kvm_timer_earliest_exp()
                queue_work_on()
                  __queue_work()
                    get_work_pool()
                    insert_work()
                      wake_up_worker()
                        wake_up_process()
                          try_to_wake_up()
                            update_rq_clock()
                            ttwu_do_activate()
                              activate_task()
                                enqueue_task_fair()
                                  enqueue_entity()
                                    update_curr()
                                    update_cfs_shares()
                                    account_entity_enqueue()
                                    __enqueue_entity()
                              wq_worker_waking_up()
                              ttwu_do_wakeup()
            __hrtimer_get_next_event()
            tick_program_event()
              clockevents_program_event()
        gic_eoimode1_eoi_irq()
    irq_exit()
      idle_cpu()
      tick_nohz_irq_exit()
      rcu_irq_exit()
//中断处理结束      
```

## core 42的正常流程
```c
cpu_pm_exit()
  cpu_pm_notify()
    rcu_irq_enter_irqson()
    __atomic_notifier_call_chain()
      notifier_call_chain()
        gic_cpu_pm_notifier()
        arch_timer_cpu_pm_notify()
        fpsimd_cpu_pm_notifier()
        cpu_pm_pmu_notify()
        hyp_init_cpu_pm_notifier()
          cpu_hyp_reinit()
          kvm_get_idmap_vector()
          kvm_mmu_get_httbr()
          kvm_arm_init_debug()
          kvm_vgic_init_cpu_hardware()
    rcu_irq_exit_irqson()      
      rcu_irq_exit()
sched_idle_set_state()
cpuidle_reflect()
rcu_idle_exit()
arch_cpu_idle_exit()
tick_nohz_idle_exit()
  ktime_get()
    arch_counter_read()
  tick_nohz_restart_sched_tick()
  account_idle_ticks()
sched_ttwu_pending()
schedule_idle() //切换到kworker-332前的准备
  rcu_note_context_switch()
  update_rq_clock()
  pick_next_task_fair()
    put_prev_task_idle()
    pick_next_entity()
    set_next_entity()
  fpsimd_thread_switch()
  hw_breakpoint_thread_switch()
  uao_thread_switch()
  qqos_sched_in()
  
<idle>-0    =>  kworker-332 //idle进程已经切换到kworker

finish_task_switch()
_raw_spin_lock_irq()
process_one_work() //kworker代码 -- 84 us
  find_worker_executing_work()
  set_work_pool_and_clear_pending()
  kvm_timer_inject_irq_work()
    kvm_vcpu_wake_up()
      swake_up()
        swake_up_locked()
          wake_up_process()
            try_to_wake_up()
  _cond_resched()
  rcu_all_qs()
  pwq_dec_nr_in_flight()
worker_enter_idle()
schedule()
  rcu_note_context_switch()
  update_rq_clock()
  deactivate_task()
    dequeue_task_fair()
      dequeue_entity()    
      hrtick_update()
  wq_worker_sleeping()
  pick_next_task_fair()
    check_cfs_rq_runtime()
    pick_next_entity()
    set_next_entity()
  fpsimd_thread_switch()
  hw_breakpoint_thread_switch()
  uao_thread_switch()
  qqos_sched_in()
  
kworker-332   =>  CPU 2/K-6915 //kworker已经切换到CPU 2/K, 后者是qemu的VCPU进程

finish_task_switch()
  kvm_sched_in()
    kvm_arch_vcpu_load()
      kvm_vgic_load()
prepare_to_swait()
kvm_vcpu_check_block()
  kvm_arch_vcpu_runnable()
  kvm_cpu_has_pending_timer()
finish_swait()  
ktime_get()
kvm_arch_vcpu_unblocking() //-- 11 us
  kvm_timer_unschedule()

//从这里开始的几个顶级函数, 是在kvm_arch_vcpu_ioctl_run()里调用的, 后者是用户态通过VCPU_RUN ioctl调用的, 它在一个循环里执行VM的代码直到时间片结束.
_cond_resched()
kvm_pmu_flush_hwstate()
  kvm_pmu_update_state()
kvm_timer_flush_hwstate() //20 us
  kvm_timer_update_state()
    kvm_timer_should_fire.part.9()
    kvm_timer_update_irq.isra.5()
      kvm_vgic_inject_irq()
        vgic_lazy_init()
        vgic_get_irq()
        vgic_queue_irq_unlock()
          vgic_target_oracle()
          kvm_vcpu_kick() //奇怪的是函数里的if (kvm_vcpu_wake_up(vcpu))为什么没出现?
  irq_set_irqchip_state()        
kvm_vgic_flush_hwstate()  
kvm_timer_should_notify_user()
kvm_pmu_should_notify_user()
kvm_pmu_sync_hwstate()
kvm_timer_sync_hwstate()
kvm_vgic_sync_hwstate()
kvm_arm_setup_debug()
//trace_kvm_entry(*vcpu_pc(vcpu)); 我自己看代码加的, 进入VM
//guest_enter_irqoff()
//对rcu来说, 执行guest代码和切换到user代码差不多
rcu_note_context_switch()
kvm_arm_clear_debug()
handle_exit()
  kvm_handle_wfx()
    kvm_vcpu_block()
      ktime_get()
      kvm_arch_vcpu_blocking()
      prepare_to_swait()
      kvm_vcpu_check_block()
      schedule() //调度到idle进程

CPU 2/K-6915  =>    <idle>-0 //切换回idle

finish_task_switch() //接着执行第40行idle进程未完成的schedule_idle(), 时间过去了467 us, idle被换出了467 us
do_idle()
  quiet_vmstat()
  tick_nohz_idle_enter()
    set_cpu_sd_state_idle()
    __tick_nohz_idle_enter()
  arch_cpu_idle_enter()
  tick_check_broadcast_expired()
  cpuidle_get_cpu_driver()
  rcu_idle_enter()
  cpuidle_not_available()
  cpuidle_select()
  call_cpuidle()
    cpuidle_enter()
      cpuidle_enter_state()
        sched_idle_set_state()
        acpi_idle_lpi_enter()
            cpu_pm_enter()
            psci_cpu_suspend_enter()
              psci_cpu_suspend()
                __invoke_psci_fn_smc() //陷入到EL3
//@ 585311.6 us idle了585ms

//从此开始新的一轮
cpu_pm_exit() 
```
