---
layout: post
title: "SyzGPT: When the fuzzer meets the LLM"
categories: 
  - fuzzing
  - exploitation
  - linux kernel
  - hacking
  - ai
  - gpt
  - LLM
---

[![Hacker AI cover photo](/assets/blog4_gpt_fuzzer/gpt_memory_hacker.jpeg)](/assets/blog4_gpt_fuzzer/gpt_memory_hacker.jpeg)


In this blog post I publish the results of a project I did during Thanksgiving.
While eating an unhealthy amount of turkey ![](https://cdn.betterttv.net/emote/63806d6cb9076d0aaebcf83f/1x) I brought up the topic of AI and warned my parents about the risks of its misuse.
One such risk being the biggest nightmare of every security practitioner -- AI lowering the bar for petty skids to hack real
systems after they got bored taking L's in Fortnite.

Security is not a solved problem and I don't see it being one in the near future (more code = more jobs).
As security researchers, our job is therefore to increase the bar, such that an adversary with a low enough budget doesn't do anything.
As I was making the case, at some point in the conversation I thought:
*What if I can use a large language model (LLM) to help me with fuzzing a linux kernel module that I have no idea of*?

# Background

Fuzzing the Linux kernel is a widely studied topic. There are various fuzzing tools, such as [Triforce AFL](https://github.com/nccgroup/TriforceAFL), [HFL](https://www.ndss-symposium.org/wp-content/uploads/2020/02/24018-paper.pdf), [Syzkaller](https://github.com/google/syzkaller), etc, the latter being the defacto state-of-the-art open-source kernel fuzzer.
Syzkaller relies on kernel experts' **manually-written** Syzlang descriptions to generate valid system calls. A description is a file written in syzkaller's language (Syzlang) that describes how to correctly invoke a systemcall, using the correct parameters, structures and dependencies. For example a `write` into a file requires one to `open` the file and use it's return value as a parameter for the earlier.

In <a href="#fig1">Fig. 1</a> I'm showing an example of a Syzlang description (called description from now on) for the `aio` module.
The description contains the following:

1. The required includes to write a valid C program. (yellow highlight)
2. The resources that need to be opened, such as file descriptors. (green highlight)
3. `struct` definitions that allow the fuzzer to accurately fuzz complex structures (yellow box)
4. `const`, `enum` defitions that restrict the fuzzer to values that make sense (pink bracket)
5. The system call defitions that allows the fuzzer to know what parameters to use at which positions and what data type they are. As shown in the figure, some parameters can be resources too. The fuzzer is made aware that it needs to initialize the resource using `io_setup` (blue highlight) by passing its integer pointer as an argument.


{% include image.html url="/assets/blog4_gpt_fuzzer/syzkaller_description_example.png" description="Description of `aio` module in syzlang." figid="1" %}

# Experimental setup

As mentioned earlier, these descriptions are written manually by kernel developers.
There have been efforts to automate the generation or even reduce these descriptions \[[1](https://www.cs.ucr.edu/%7Ezhiyunq/pub/oakland23_syzdescribe.pdf),[2](https://dl.acm.org/doi/10.1145/3133956.3134069),[3](https://www.usenix.org/conference/atc22/presentation/sun),[4](https://www.ndss-symposium.org/wp-content/uploads/2023/02/ndss2023_f688_paper.pdf), [5](https://dl.acm.org/doi/10.1145/3133956.3134103)\], which is beyond the scope of this blog post (but an interesting followup nonetheless). 
Regardless they all come with some built-in assumptions, and wouldn't generalize well for a random kernel subsystem. My assumption instead is that a kernel subsystem has a well-written documentation that instructs other non-kernel developers how to use it's exposed system calls to interact with it.
With this assumption in mind I hypothesize that a Large Language Model (referred as LLM from now on) can read it and understand how to translate that human-readable documentation (ie. a massive `HTML` page) into a description for Syzkaller.
To test this assumption I picked a fairly complex subsystem, the `KVM`. This subsystem has a rich description in Syzlang, clocking more than `2000` lines.
<!--  For a more complete analysis one may need to test this assumption in other subsystems and see if there is a correlation between number of defined struct, number of flags or number of lines of code to the coverage. -->

In summary, the experiment is as follows. First, I chose to enable fuzzing on the first 40 system calls from [`dev_kvm.txt`](https://github.com/google/syzkaller/blob/master/sys/linux/dev_kvm.txt), such as `openat$kvm`, `ioctl$KVM_CREATE_VM`, etc.
Second, I ran the fuzzer for **9** hours using the original manually written descriptions.
Third, I found the documentation of KVM module from [this article](https://www.kernel.org/doc/html/latest/virt/kvm/api.html) and fed it to Bing AI.
Fourth, I removed everything from `dev_kvm.txt` and populated it with what the AI wrote to me.
At last, I ran the fuzzer again for **9** hours using the AI-written descriptions.

# A chat with the quickest reader

In fourth step, I chat back and forth with Bing AI. First I primed it by making sure that it reads the web page (that contains the `KVM` documentation) and answers using that as the reference.
I believe Bing AI is using a form of information retrieval system on top of the LLM as in here [\[5\]](https://arxiv.org/pdf/2308.07107.pdf), which makes it such that the queries are always referred to text from the documentation page.
I opened the tab on the side using the Microsoft edge's [Bing button](https://www.microsoft.com/en-us/edge/learning-center/how-to-use-bing-in-sidebar?form=MA13I2) ans started asking the qeustion.
In the figure below (<a href="#fig2">Fig. 2</a>), I am showing the first message I sent to Bing AI using *GPT-4*. I asked it to tell me what it reads and see if it can interpret the web page. As it can be seen, the AI was fully capable of reading the article and summarize the syscalls in **4** classes of system calls. This is not a simple rip-off of the original summary, because the part underlined in red is not there in the general description section of the documentation. This means that the GPT was able to read though each of the individual system calls and conclude that `check extension` is a feature of the system *ioctl*s. As also shown in the figure there are plenty of other cases where the GPT is able to provide more concrete examples of each class of syscalls. In the *device ioctls* however it fails to mention that they can only be called by the process that created the VM, but it's not a big deal in my opinion.

{% include image.html url="/assets/blog4_gpt_fuzzer/chat_start.png" description="Priming the LLM" figid="2" %}

Next, I asked the GPT to generate a program that uses `KVM_CREATE_VM` and it wrote it in C. To save some space, I am not posting the interactions that were needed to ask it to write the `C` program. I then tested it to make sure it understood how `Syzlang` works and whether it can translate the `KVM_CREATE_VM` syscall back into `Syzlang`. The response is shown in <a href="#fig3">Fig. 3</a>. As you can see, the model is capable of understanding how `Syzlang` works and what information from a C program are required by it. Additionally it not only wrote the syscall I asked it, but also the syscall `ioctl$KVM_CREATE_VM` is dependent on, `openat$KVM`.

{% include image.html url="/assets/blog4_gpt_fuzzer/chat_testing.png" description="Testing LLM's capability to convert C to Syzlang" figid="3" %}

# We need to go deeper...

As you can see above the `ioctl$KVM_CREATE_VM` syscall is correct, but it's a specific one, so it cannot be used to fuzz, because none of the parameters can be modified by the fuzzer. So naturally, I asked GPT to make a generic version of this system call. 
In <a href="#fig4">Fig. 4</a> you can see the output of GPT. It's asstounding how it not only wrote a generic `Syzlang` representation but also explained every parameter, the dependencies and all the possible flag values and the header where they are defined. I was shocked at how well it described the system call and how deep was it's understanding.
This were my emotions when I read this output ![](https://cdn.betterttv.net/emote/5f664c989d79b403d93387cc/1x.webp) ![](https://cdn.betterttv.net/emote/58f6e05e58f5dd226a16166e/1x.webp).

{% include image.html url="/assets/blog4_gpt_fuzzer/1_syscall_try.png" description="Writing the Syzlang description for <i>openat$KVM_CREATE_VM</i>" figid="4" %}

Well, now the genie is out of the bottle. Let's just ask our overlord for more blessings. In <a href="#fig5">Fig. 5</a> you can see our conversation. For the sake of brevity, I won't be posting the entire result, only the beginning of it. The GPT hit the word limit and it didn't generate all the syscall descriptions.

{% include image.html url="/assets/blog4_gpt_fuzzer/many_syscalls.png" description="Making GPT generate syzlang representation of more syscalls" figid="5" %}

As I showed earlier there are 4 things in a description file: (1) the includes, (2) the resources, (3) the syscalls and (4) the struct and flag definitions. We have tackled the first 3. Now let's ask GPT to tackle the 4th too and bundle up all into 1 so that we can copy and paste.
In the same conversation I asked GPT:

<div id="enabled_syscalls"></div>
```
write me a syzkaller description with all the kvm syscalls mentioned below:


openat$kvm
openat$sgx_provision
ioctl$KVM_CREATE_VM
ioctl$KVM_GET_MSR_INDEX_LIST
ioctl$KVM_CHECK_EXTENSION
ioctl$KVM_GET_VCPU_MMAP_SIZE
ioctl$KVM_GET_SUPPORTED_CPUID
ioctl$KVM_GET_EMULATED_CPUID
ioctl$KVM_X86_GET_MCE_CAP_SUPPORTED
ioctl$KVM_GET_API_VERSION
ioctl$KVM_CREATE_VCPU
ioctl$KVM_CHECK_EXTENSION_VM
ioctl$KVM_GET_DIRTY_LOG
ioctl$KVM_CREATE_IRQCHIP
ioctl$KVM_IRQ_LINE
ioctl$KVM_IRQ_LINE_STATUS
ioctl$KVM_GET_IRQCHIP
ioctl$KVM_SET_IRQCHIP
ioctl$KVM_XEN_HVM_CONFIG
ioctl$KVM_GET_CLOCK
ioctl$KVM_SET_CLOCK
ioctl$KVM_SET_USER_MEMORY_REGION
ioctl$KVM_SET_TSS_ADDR
ioctl$KVM_SET_IDENTITY_MAP_ADDR
ioctl$KVM_SET_BOOT_CPU_ID
ioctl$KVM_PPC_GET_PVINFO
ioctl$KVM_ASSIGN_PCI_DEVICE
ioctl$KVM_DEASSIGN_PCI_DEVICE
ioctl$KVM_ASSIGN_DEV_IRQ
ioctl$KVM_DEASSIGN_DEV_IRQ
ioctl$KVM_SET_GSI_ROUTING
ioctl$KVM_ASSIGN_SET_MSIX_NR
ioctl$KVM_ASSIGN_SET_MSIX_ENTRY
ioctl$KVM_IOEVENTFD
ioctl$KVM_ASSIGN_SET_INTX_MASK
ioctl$KVM_SIGNAL_MSI
ioctl$KVM_CREATE_PIT2
ioctl$KVM_GET_PIT
ioctl$KVM_SET_PIT
ioctl$KVM_GET_PIT2
ioctl$KVM_SET_PIT2
ioctl$KVM_PPC_GET_SMMU_INFO
ioctl$KVM_IRQFD
ioctl$KVM_PPC_ALLOCATE_HTAB
ioctl$KVM_CREATE_DEVICE
ioctl$KVM_REGISTER_COALESCED_MMIO
ioctl$KVM_UNREGISTER_COALESCED_MMIO
ioctl$KVM_SET_NR_MMU_PAGES
ioctl$KVM_GET_NR_MMU_PAGES
ioctl$KVM_REINJECT_CONTROL
ioctl$KVM_HYPERV_EVENTFD
ioctl$KVM_CAP_MANUAL_DIRTY_LOG_PROTECT2
ioctl$KVM_CAP_HALT_POLL
ioctl$KVM_CAP_DIRTY_LOG_RING
```

With this prompt GPT replied as follows:

```
include <linux/kvm.h>
include <linux/kvm_para.h>
include <linux/kvm_types.h>

resource fd_kvm[fd]
resource fd_kvm_vm[fd]
resource fd_kvm_vcpu[fd]
resource fd_kvm_device[fd]

openat$kvm(fd const[AT_FDCWD], file ptr[in, string["/dev/kvm"]], flags flags[open_flags], mode const[0]) fd_kvm
openat$sgx_provision(fd const[AT_FDCWD], file ptr[in, string["/dev/sgx/provision"]], flags flags[open_flags], mode const[0]) fd_kvm

ioctl$KVM_CREATE_VM(fd fd_kvm, cmd const[KVM_CREATE_VM], arg const[0]) fd_kvm_vm
ioctl$KVM_GET_MSR_INDEX_LIST(fd fd_kvm, cmd const[KVM_GET_MSR_INDEX_LIST], arg ptr[out, kvm_msr_list])
ioctl$KVM_CHECK_EXTENSION(fd fd_kvm, cmd const[KVM_CHECK_EXTENSION], arg int32[kvm_extensions])
ioctl$KVM_GET_VCPU_MMAP_SIZE(fd fd_kvm, cmd const[KVM_GET_VCPU_MMAP_SIZE], arg const[0])
ioctl$KVM_GET_SUPPORTED_CPUID(fd fd_kvm, cmd const[KVM_GET_SUPPORTED_CPUID], arg ptr[inout, kvm_cpuid2])
ioctl$KVM_GET_EMULATED_CPUID(fd fd_kvm, cmd const[KVM_GET_EMULATED_CPUID], arg ptr[inout, kvm_cpuid2])
ioctl$KVM_X86_GET_MCE_CAP_SUPPORTED(fd fd_kvm, cmd const[KVM_X86_GET_MCE_CAP_SUPPORTED], arg ptr[out, kvm_x86_mce])
ioctl$KVM_GET_API_VERSION(fd fd_kvm, cmd const[KVM_GET_API_VERSION], arg const[0])

ioctl$KVM_CREATE_VCPU(fd fd_kvm_vm, cmd const[KVM_CREATE_VCPU], arg int32) fd_kvm_vcpu
ioctl$KVM_CHECK_EXTENSION_VM(fd fd_kvm_vm, cmd const[KVM_CHECK_EXTENSION], arg int32[kvm_extensions])
ioctl$KVM_GET_DIRTY_LOG(fd fd_kvm_vm, cmd const[KVM_GET_DIRTY_LOG], arg ptr[in, kvm_dirty_log])
ioctl$KVM_CREATE_IRQCHIP(fd fd_kvm_vm, cmd const[KVM_CREATE_IRQCHIP], arg const[0])
ioctl$KVM_IRQ_LINE(fd fd_kvm_vm, cmd const[KVM_IRQ_LINE], arg ptr[in, kvm_irq_level])
ioctl$KVM_IRQ_LINE_STATUS(fd fd_kvm_vm, cmd const[KVM_IRQ_LINE_STATUS], arg ptr[inout, kvm_irq_level])
ioctl$KVM_GET_IRQCHIP(fd fd_kvm_vm, cmd const[KVM_GET_IRQCHIP], arg ptr[inout, kvm_irqchip])
ioctl$KVM_SET_IRQCHIP(fd fd_kvm_vm, cmd const[KVM_SET_IRQCHIP], arg ptr[in, kvm_irqchip])
ioctl$KVM_XEN_HVM_CONFIG(fd fd_kvm_vm, cmd const[KVM_XEN_HVM_CONFIG], arg ptr[in, kvm_xen_hvm_config])
ioctl$KVM_GET_CLOCK(fd fd_kvm_vm, cmd const[KVM_GET_CLOCK], arg ptr[out, kvm_clock_data])
ioctl$KVM_SET_CLOCK(fd fd_kvm_vm, cmd const[KVM_SET_CLOCK], arg ptr[in, kvm_clock_data])
ioctl$KVM_SET_USER_MEMORY_REGION(fd fd_kvm_vm, cmd const[KVM_SET_USER_MEMORY_REGION], arg ptr[in, kvm_userspace_memory_region])
ioctl$KVM_SET_TSS_ADDR(fd fd_kvm_vm, cmd const[KVM_SET_TSS_ADDR], arg int64)
ioctl$KVM_SET_IDENTITY_MAP_ADDR(fd fd_kvm_vm, cmd const[KVM_SET_IDENTITY_MAP_ADDR], arg int64)
ioctl$KVM_SET_BOOT_CPU_ID(fd fd_kvm_vm, cmd const[KVM_SET_BOOT_CPU_ID], arg int32)
ioctl$KVM_PPC_GET_PVINFO(fd fd_kvm_vm, cmd const[KVM_PPC_GET_PVINFO], arg ptr[out, kvm_ppc_pvinfo])
ioctl$KVM_ASSIGN_PCI_DEVICE(fd fd_kvm_vm, cmd const[KVM_ASSIGN_PCI_DEVICE], arg ptr[in, kvm_assigned_pci_dev])
ioctl$KVM_DEASSIGN_PCI_DEVICE(fd fd_kvm_vm, cmd const[KVM_DEASSIGN_PCI_DEVICE], arg ptr[in, kvm_assigned_pci_dev])
ioctl$KVM_ASSIGN_DEV_IRQ(fd fd_kvm_vm, cmd const[KVM_ASSIGN_DEV_IRQ], arg ptr[in, kvm_assigned_irq])
ioctl$KVM_DEASSIGN_DEV_IRQ(fd fd_kvm_vm, cmd const[KVM_DEASSIGN_DEV_IRQ], arg ptr[in, kvm_assigned_irq])
ioctl$KVM_SET_GSI_ROUTING(fd fd_kvm_vm, cmd const[KVM_SET_GSI_ROUTING], arg ptr[in, kvm_irq_routing])
ioctl$KVM_ASSIGN_SET_MSIX_NR(fd fd_kvm_vm, cmd const[KVM_ASSIGN_SET_MSIX_NR], arg ptr[in, kvm_assigned_msix_nr])
ioctl$KVM_ASSIGN_SET_MSIX_ENTRY(fd fd_kvm_vm, cmd const[KVM_ASSIGN_SET_MSIX_ENTRY], arg ptr[in, kvm_assigned_msix_entry])
ioctl$KVM_IOEVENTFD(fd fd_kvm_vm, cmd const[KVM_IOEVENTFD], arg ptr[in, kvm_ioeventfd])
ioctl$KVM_ASSIGN_SET_INTX_MASK(fd fd_kvm_vm, cmd const[KVM_ASSIGN_SET_INTX_MASK], arg ptr[in, kvm_assigned_pci_dev])
ioctl$KVM_SIGNAL_MSI(fd fd_kvm_vm, cmd const[KVM_SIGNAL_MSI], arg ptr[in, kvm_msi])
ioctl$KVM_CREATE_PIT2(fd fd_kvm_vm, cmd const[KVM_CREATE_PIT2], arg ptr[in, kvm_pit_config])
ioctl$KVM_GET_PIT(fd fd_kvm_vm, cmd const[KVM_GET_PIT], arg ptr[out, kvm_pit_state])
ioctl$KVM_SET_PIT(fd fd_kvm_vm, cmd const[KVM_SET_PIT], arg ptr[in, kvm_pit_state])
ioctl$KVM_GET_PIT2(fd fd_kvm_vm, cmd const[KVM_GET_PIT2], arg ptr[out, kvm_pit_state2])
ioctl$KVM_SET_PIT2(fd fd_kvm_vm, cmd const[KVM_SET_PIT2], arg ptr[in, kvm_pit_state2])
ioctl$KVM_PPC_GET_SMMU_INFO(fd fd_kvm_vm, cmd const[KVM_PPC_GET_SMMU_INFO], arg ptr[out, kvm_ppc_smmu_info])
ioctl$KVM_IRQFD(fd fd_kvm_vm, cmd const[KVM_IRQFD], arg ptr[in, kvm_irqfd])
ioctl$KVM_PPC_ALLOCATE_HTAB(fd fd_kvm_vm, cmd const[KVM_PPC_ALLOCATE_HTAB], arg int32)
ioctl$KVM_CREATE_DEVICE(fd fd_kvm_vm, cmd const[KVM_CREATE_DEVICE], arg ptr[inout, kvm_create_device]) fd_kvm_device
ioctl$KVM_REGISTER_COALESCED_MMIO(fd fd_kvm_vm, cmd const[KVM_REGISTER_COALESCED_MMIO], arg ptr[in, kvm_coalesced_mmio_zone])
ioctl$KVM_UNREGISTER_COALESCED_MMIO(fd fd_kvm_vm, cmd const[KVM_UNREGISTER_COALESCED_MMIO], arg ptr[in, kvm_coalesced_mmio_zone])
ioctl$KVM_SET_NR_MMU_PAGES(fd fd_kvm_vm, cmd const[KVM_SET_NR_MMU_PAGES], arg int32)
ioctl$KVM_GET_NR_MMU_PAGES(fd fd_kvm_vm, cmd const[KVM_GET_NR_MMU_PAGES], arg const[0])
ioctl$KVM_REINJECT_CONTROL(fd fd_kvm_vm, cmd const[KVM_REINJECT_CONTROL], arg int32)
ioctl$KVM_HYPERV_EVENTFD(fd fd_kvm_vm, cmd const[KVM_HYPERV_EVENTFD], arg ptr[in, kvm_hyperv_eventfd])
ioctl$KVM_CAP_MANUAL_DIRTY_LOG_PROTECT2(fd fd_kvm_vm, cmd const[KVM_CAP_MANUAL_DIRTY_LOG_PROTECT2], arg int32)
ioctl$KVM_CAP_HALT_POLL(fd fd_kvm_vm, cmd const[KVM_CAP_HALT_POLL], arg int32)
ioctl$KVM_CAP_DIRTY_LOG_RING(fd fd_kvm_vm, cmd const[KVM_CAP_DIRTY_LOG_RING], arg int32)

kvm_extensions = KVM_CAP_IRQCHIP, KVM_CAP_HLT, KVM_CAP_MMU_SHADOW_CACHE_CONTROL, KVM_CAP_USER_MEMORY, KVM_CAP_SET_TSS_ADDR, KVM_CAP_VAPIC, KVM_CAP_EXT_CPUID, KVM_CAP_CLOCKSOURCE, KVM_CAP_NR_VCPUS, KVM_CAP_NR_MEMSLOTS, KVM_CAP_PIT, KVM_CAP_NOP_IO_DELAY, KVM_CAP_PV_MMU, KVM_CAP_MP_STATE, KVM_CAP_COALESCED_MMIO, KVM_CAP_SYNC_MMU, KVM_CAP_IOMMU, KVM_CAP_DESTROY_MEMORY_REGION_WORKS, KVM_CAP_USER_NMI, KVM_CAP_SET_GUEST_DEBUG, KVM_CAP_REINJECT_CONTROL, KVM_CAP_IRQ_ROUTING, KVM_CAP_IRQ_INJECT_STATUS, KVM_CAP_ASSIGN_DEV_IRQ, KVM_CAP_JOIN_MEMORY_REGIONS_WORKS, KVM_CAP_MCE, KVM_CAP_IRQFD, KVM_CAP_PIT2, KVM_CAP_SET_BOOT_CPU_ID, KVM_CAP_PIT_STATE2, KVM_CAP
```

As you can see the the definition of the structs is still missing. GPT ran out of words before finishing typing the whole file.
So I asked it to `write the struct definitions such that the system calls you wrote above can be compiled`. And GPT replied the `struct` definitions.
I noticed that these structs were explicity in the KVM documentation but GPT was smart enough to know how to convert kernel-defined data types to the ones defined by Syzkaller or the userland include clauses. At the end of the post I will link the full GPT-generated description


# Man against machine

I pulled syzkaller commit [8321139737ed27c](https://github.com/google/syzkaller/commit/8321139737ed27c) in 2 different folders. In one I compiled it and in the other I replaced `/sys/linux/dev_kvm.txt` with the one generated via Bing AI. Then I let both run on the system calls I mentioned <a href="#enabled_syscalls">here</a>. I let both run for around **9** hours, on 1 VM using `--debug` flag of `syz-manager`.


{% include image.html url="/assets/blog4_gpt_fuzzer/manual_vs_gpt.png" description="Fuzzer stats of manually generated(left) VS GPT-generated(right) descriptions." figid="6" %}

In the <a href="#fig6">Fig. 6</a> I'm reporting the results of running both fuzzers. As you cna see in the left, the manually-written system calls showed a coverage of `6536` code blocks when it ran for `9h50min` while the GPT-generated one showed a coverage of `5659` when it ran for `9h17min`. The runtime is not exactly the same but I saw the fuzzers started to plateau. Additionally, the manually generated descriptions showed a max signal of `16813` while the GPT-generated one showed only `14110`. Either way it took me less than 30 min with GPT to generate `dev_kvm.txt`. Did mankind win this ![](https://cdn.betterttv.net/emote/5aa16eb65d4a424654d7e3e5/1x.webp) ? I will leave it up to your judgement.

# Conclusion

To sum up, in this blog post I briefly evaluate the ability of a LLM to automatically understand a kernel subsystem's documentation to generate *Syzlang* descriptions, to guide the kernel fuzzer. The results show that for a subset of syscalls of `KVM` the manually-written rules outperform the kernel fuzzer. This result is not all-encompassing - other kernel subsystems may show different results. Additionally, this method assumes that a well structured and detailed documnetation on how to call the API is present. This might not always be the case. Nonetheless, GPT demonstrated a significant understanding of the `KVM` subsystem and *Syzlang* and I it not only shortened the time to write a description, but also made it more accessible for a kernel developer with limited knowledge of Syzlang to write one for their own kernel module. To support open science here is the [`dev_kvm.txt`](/assets/blog4_gpt_fuzzer/dev_kvm.txt) file that that the GPT helped generate.