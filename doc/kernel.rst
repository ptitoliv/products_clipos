.. Copyright © 2018 ANSSI.
   CLIP OS is a trademark of the French Republic.
   Content licensed under the Open License version 2.0 as published by Etalab
   (French task force for Open Data).

.. _kernel:

Kernel
======

The CLIP OS kernel is based on Linux. It also integrates:

* existing hardening patches that are not upstream yet and that we consider
  relevant to our security model;
* developments made for previous CLIP OS versions that we have not upstreamed
  yet (or that cannot be);
* entirely new functionalities that have not been upstreamed yet (or that
  cannot be).

Objectives
----------

As the core of a hardened operating system, the CLIP OS kernel is particularly
responsible for:

* providing **robust security mechanisms** to higher levels of the operating
  system, such as reliable isolation primitives;
* maintaining maximal **trust in hardware resources**;
* guaranteeing its **own protection** against various threats.

Configuration
-------------

In this section we discuss our security-relevant configuration choices for
the CLIP OS kernel. Before starting, it is worth mentioning that:

* We do our best to **limit the number of kernel modules**.

  In other words, as many modules as possible should be built-in. Modules are
  only used when needed either for the initramfs or to ease the automation of
  the deployment of CLIP OS on multiple different machines (for the moment, we
  only target a QEMU-KVM guest). This is particularly important as module
  loading will be disabled after CLIP OS startup.

* We **focus on a secure configuration**. The remaining of the configuration
  is minimal and it is your job to tune it for your machines and use cases.

* CLIP OS only supports the x86-64 architecture for now.

* Running 32-bit programs is voluntarily unsupported. Should you change that
  in your custom kernel, keep in mind that it requires further attention when
  configuring it (e.g., ensure that ``CONFIG_COMPAT_VDSO=n``).

* Many options that are not useful to us are disabled in order to cut attack
  surface. As they are not all detailed below, please see
  ``src/portage/clip/sys-kernel/clipos-kernel/files/config.d/blacklist`` for an
  exhaustive list of the ones we **explicitly** disable.

General setup
~~~~~~~~~~~~~

.. describe:: CONFIG_AUDIT=y

   CLIP OS will need the auditing infrastructure.

.. describe:: CONFIG_IKCONFIG=n

   We do not need ``.config`` to be available at runtime.

.. describe:: CONFIG_KALLSYMS=n

   Symbols are only useful for debug and attack purposes.

.. describe:: CONFIG_EXPERT=y

   This unlocks additional configuration options we need.

.. ---

.. describe:: CONFIG_USER_NS=n

   User namespaces can be useful for some use cases but even more to an
   attacker. We choose to disable them for the moment, but we could also enable
   them and use the ``kernel.unprivileged_userns_clone`` sysctl provided by
   linux-hardened to disable their unprivileged use.

.. ---

.. describe:: CONFIG_SLUB_DEBUG=y

   Allow allocator validation checking to be enabled.

.. describe:: CONFIG_SLAB_MERGE_DEFAULT=n

   Merging SLAB caches can make heap exploitation easier.

.. describe:: CONFIG_SLAB_FREELIST_RANDOM=y

   Randomize allocator freelists

.. describe:: CONFIG_SLAB_FREELIST_HARDENED=y

   Harden slab metadata

.. describe:: CONFIG_SLAB_HARDENED=y

   Add various little checks to harden the slab allocator. [linux-hardened]_

.. describe:: CONFIG_SLAB_CANARY=y

   Place canaries at the end of slab allocations. [linux-hardened]_

.. describe:: CONFIG_SLAB_SANITIZE=y

   Zero-fill slab allocations on free to reduce risks of information leaks and
   help mitigate use-after-free vulnerabilities. [linux-hardened]_

   .. describe:: CONFIG_SLAB_SANITIZE_VERIFY=y

      Verify that newly allocated slab allocations are zeroed to detect
      write-after-free bugs. [linux-hardened]_


.. ---

.. describe:: CONFIG_COMPAT_BRK=n

   Enabling this would disable brk ASLR.

.. ---

.. describe:: CONFIG_GCC_PLUGINS=y

   Enable GCC plugins, some of which are security-relevant; GCC 4.7 at least is
   required.

   .. describe:: CONFIG_GCC_PLUGIN_LATENT_ENTROPY=y

      Instrument some kernel code to gather additional (but not
      cryptographically secure) entropy at boot time.

   .. describe:: CONFIG_GCC_PLUGIN_STRUCTLEAK=y

      Prevent potential information leakage by forcing initialization of
      structures containing userspace addresses. This is particularly
      important to prevent trivial bypassing of KASLR.

   .. describe:: CONFIG_GCC_PLUGIN_STRUCTLEAK_BYREF_ALL=y

      Extend forced initialization to all local structures that have their
      address taken at any point.

   .. describe:: CONFIG_GCC_PLUGIN_RANDSTRUCT=y

      Randomize layout of sensitive kernel structures. Exploits targeting such
      structures then require an additional information leak vulnerability.

   .. describe:: CONFIG_GCC_PLUGIN_RANDSTRUCT_PERFORMANCE=n

      Do not weaken structure randomization

.. ---

.. describe:: CONFIG_ARCH_MMAP_RND_BITS=32

   Use maximum number of randomized bits for the mmap base address on x86_64.
   Note that thanks to a linux-hardened patch, this also impacts the number of
   randomized bits for the stack base address.

.. ---

.. describe:: CONFIG_STACKPROTECTOR=y
              CONFIG_STACKPROTECTOR_STRONG=y

   Use ``-fstack-protector-strong`` for best stack canary coverage; GCC 4.9 at
   least is required.

.. describe:: CONFIG_VMAP_STACK=y

   Virtually-mapped stacks benefit from guard pages, thus making kernel stack
   overflows harder to exploit.

.. describe:: CONFIG_REFCOUNT_FULL=y

   Do extensive checks on reference counting to prevent use-after-free
   conditions. Without this option, on x86, there already is a fast
   assembly-based protection based on the PaX implementation but it does not
   cover all cases.

.. ---

.. describe:: CONFIG_STRICT_MODULE_RWX=y

   Enforce strict memory mappings permissions for loadable kernel modules.

.. ---

Although CLIP OS stores kernel modules in an initramfs located in a signed
EFI binary (together with the kernel itself and its command line), we still
enable and enforce module signing as an additional layer of security:

 .. describe:: CONFIG_MODULE_SIG=y
               CONFIG_MODULE_SIG_FORCE=y
               CONFIG_MODULE_SIG_ALL=y
               CONFIG_MODULE_SIG_SHA512=y
               CONFIG_MODULE_SIG_HASH="sha512"

.. ---

.. describe:: CONFIG_LOCAL_INIT=n

   This option requires compiler support for ``-fsanitize=local-init``, which
   is only available in Clang. [linux-hardened]_

Processor type and features
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. describe:: CONFIG_RETPOLINE=y

   Retpolines are needed to protect against Spectre v2. GCC 7.3.0 or higher is
   required.

.. describe:: CONFIG_LEGACY_VSYSCALL_NONE=y

   The vsyscall table is not required anymore by libc and is a fixed-position
   potential source of ROP gadgets.

.. describe:: CONFIG_X86_VSYSCALL_EMULATION=n

   See above.

.. describe:: CONFIG_MICROCODE=y

   Needed to benefit from microcode updates and thus security fixes (e.g.,
   additional Intel pseudo-MSRs to be used by the kernel as a mitigation for
   various speculative execution vulnerabilities).

.. describe:: CONFIG_X86_MSR=y

   See above explanation about ``CONFIG_MICROCODE``.

.. describe:: CONFIG_KSM=n

   Enabling this feature can make cache side-channel attacks such as
   FLUSH+RELOAD much easier to carry out.

.. ---

.. describe:: CONFIG_DEFAULT_MMAP_MIN_ADDR=65536

   This should in particular be non-zero to prevent the exploitation of kernel
   NULL pointer bugs.

.. describe:: CONFIG_MTRR=y

   Memory Type Range Registers can make speculative execution bugs a bit harder
   to exploit.

.. describe:: CONFIG_X86_PAT=y

   Page Attribute Tables are the modern equivalents of MTRRs, which we
   described above.

.. describe:: CONFIG_ARCH_RANDOM=y

   Enable the RDRAND instruction to benefit from a secure hardware RNG if
   supported. See ``CONFIG_RANDOM_TRUST_CPU`` for warnings about that.

.. describe:: CONFIG_X86_SMAP=y

   Enable Supervisor Mode Access Prevention to prevent ret2usr exploitation
   techniques.

.. describe:: CONFIG_X86_INTEL_UMIP=y

   Enable User Mode Instruction Prevention. Note that hardware supporting this
   feature is not common yet.

.. describe:: CONFIG_X86_INTEL_MPX=n

   Intel Memory Protection Extensions add hardware assistance to memory
   protection. Compiler support is required but is deprecated in GCC 8 and will
   probably be dropped in GCC 9.

.. describe:: CONFIG_X86_INTEL_MEMORY_PROTECTION_KEYS=n

   Memory Protection Keys are a promising feature but they are still not
   supported on current hardware.

.. ---

Enable the **seccomp** BPF userspace API for syscall attack surface reduction:

  .. describe:: CONFIG_SECCOMP=y
                CONFIG_SECCOMP_FILTER=y

.. ---

.. describe:: CONFIG_RANDOMIZE_BASE=y

   While this may be seen as a `controversial
   <https://grsecurity.net/kaslr_an_exercise_in_cargo_cult_security.php>`_
   feature, it makes sense for CLIP OS. Indeed, KASLR may be defeated thanks to
   the kernel interfaces that are available to an attacker, or through attacks
   leveraging hardware vulnerabilities such as speculative and out-of-order
   execution ones. However, CLIP OS follows the *defense in depth* principle
   and an attack surface reduction approach. Thus, the following points make
   KASLR relevant in the CLIP OS kernel:

   * KASLR was initially designed to counter remote attacks but the strong
     security model of CLIP OS (e.g., no sysfs mounts in most containers,
     minimal procfs, no arbitrary code execution) makes a local attack
     more complex to carry out.
   * STRUCTLEAK, STACKLEAK, kptr_restrict and
     ``CONFIG_SECURITY_DMESG_RESTRICT`` are enabled in CLIP OS.
   * The CLIP OS kernel is custom-compiled (at least for a given deployment),
     its image is unreadable to all users including privileged ones and updates
     are end-to-end encrypted. This makes both the content and addresses of the
     kernel image secret. Note that, however, the production kernel image is
     currently part of an EFI binary and is not encrypted, causing it to be
     accessible to a physical attacker. This will change in the future as we
     will only use the kernel included in the EFI binary to boot and then
     *kexec* to the real production kernel whose image will be located on an
     encrypted disk partition.
   * We enable ``CONFIG_PANIC_ON_OOPS`` by default so that the kernel
     cannot recover from failed exploit attempts, thus preventing any brute
     forcing.
   * We enable Kernel Page Table Isolation, mitigating Meltdown and potential
     other hardware information leakage. Variante 3a (Rogue System Register
     Read) however remains an important threat to KASLR.

.. ---

.. describe:: CONFIG_RANDOMIZE_MEMORY=y

   Most of the above explanations stand for that feature.

.. describe:: CONFIG_KEXEC=n
              CONFIG_KEXEC_FILE=n

   Disable the ``kexec()`` system call to prevent an already-root attacker from
   rebooting on an untrusted kernel.

.. describe:: CONFIG_CRASH_DUMP=n

   A crash dump can potentially provide an attacker with useful information.
   However we disabled ``kexec()`` syscalls above thus this configuration
   option should have no impact anyway.

.. ---

.. describe:: CONFIG_MODIFY_LDT_SYSCALL=n

   This is not supposed to be needed by userspace applications and only
   increases the kernel attack surface.

Power management and ACPI options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. describe:: CONFIG_HIBERNATION=n

   The CLIP OS swap partition is encrypted with an ephemeral key and thus
   cannot support suspend to disk.

Executable file formats / Emulations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. describe:: CONFIG_BINFMT_MISC=n

   We do not want our kernel to support miscellaneous binary classes. ELF
   binaries and interpreted scripts starting with a shebang are enough.

.. describe:: CONFIG_COREDUMP=n

   Core dumps can provide an attacker with useful information.

Networking support
~~~~~~~~~~~~~~~~~~

.. describe:: CONFIG_SYN_COOKIES=y

   Enable TCP syncookies.

Device Drivers
~~~~~~~~~~~~~~

.. describe:: CONFIG_TCG_TPM=n

   TPM use is not supported by CLIP OS yet.

.. describe:: CONFIG_DEVMEM=n

   The ``/dev/mem`` device should not be required by any user application
   nowadays.

   .. note::

      If you must enable it, at least enable ``CONFIG_STRICT_DEVMEM`` and
      ``CONFIG_IO_STRICT_DEVMEM`` to restrict at best access to this device.

.. describe:: CONFIG_DEVKMEM=n

   This virtual device is only useful for debug purposes and is very dangerous
   as it allows direct kernel memory writing (particularly useful for
   rootkits).

.. describe:: CONFIG_LEGACY_PTYS=n

   Use the modern PTY interface only.

.. describe:: CONFIG_DEVPORT=n

   The ``/dev/port`` device should not be used anymore by userspace, and it
   could increase the kernel attack surface.

.. describe:: CONFIG_RANDOM_TRUST_CPU=n

   Do not rely exclusively on the hardware RNG provided by the CPU manufacturer
   to initialize Linux's CRNG, as we do not mind blocking a bit more at boot
   time while additional entropy sources are mixed in.

The IOMMU allows for protecting the system's main memory from arbitrary
accesses from devices (e.g., DMA attacks). Note that this is related to
hardware features. On a recent Intel machine, we enable the following:

  .. describe:: CONFIG_IOMMU_SUPPORT=y
                CONFIG_INTEL_IOMMU=y
                CONFIG_INTEL_IOMMU_SVM=y
                CONFIG_INTEL_IOMMU_DEFAULT_ON=y

File systems
~~~~~~~~~~~~

.. describe:: CONFIG_PROC_KCORE=n

   Enabling this would provide an attacker with precious information on the
   running kernel.

Kernel hacking
~~~~~~~~~~~~~~

.. describe:: CONFIG_MAGIC_SYSRQ=n

   This should only be needed for debugging.

.. describe:: CONFIG_DEBUG_KERNEL=y

   This is useful even in a production kernel to enable further configuration
   options that have security benefits.

.. describe:: CONFIG_DEBUG_VIRTUAL=y

   Enable sanity checks in virtual to page code.

.. describe:: CONFIG_STRICT_KERNEL_RWX=y

   Ensure kernel page tables have strict permissions.

.. describe:: CONFIG_DEBUG_WX=y

   Check and report any dangerous memory mapping permissions, i.e., both
   writable and executable kernel pages.

.. describe:: CONFIG_DEBUG_FS=n

   The debugfs virtual file system is only useful for debugging and protecting
   it would require additional work.

.. describe:: CONFIG_SLUB_DEBUG_ON=n

   Using the ``slub_debug`` command line parameter provides more fine grained
   control.

.. describe:: CONFIG_PANIC_ON_OOPS=y
              CONFIG_PANIC_TIMEOUT=-1

   Prevent potential further exploitation of a bug by immediately panicking the
   kernel.

The following options add additional checks and validation for various
commonly targeted kernel structures:

  .. describe:: CONFIG_DEBUG_CREDENTIALS=y
                CONFIG_DEBUG_NOTIFIERS=y
                CONFIG_DEBUG_LIST=y
                CONFIG_DEBUG_SG=y
  .. describe:: CONFIG_BUG_ON_DATA_CORRUPTION=y

     Note that linux-hardened patches add more places where this configuration
     option has an impact.

  .. describe:: CONFIG_SCHED_STACK_END_CHECK=y
  .. describe:: CONFIG_PAGE_POISONING=n

     We choose to poison pages with zeroes and thus prefer using the simpler
     PaX-based implementation provided by linux-hardened (see
     ``CONFIG_PAGE_SANITIZE`` below).

Security
~~~~~~~~

.. describe:: CONFIG_SECURITY_DMESG_RESTRICT=y

   Prevent unprivileged users from gathering information from the kernel log
   buffer via ``dmesg(8)``. Note that this still can be overridden through the
   ``kernel.dmesg_restrict`` sysctl.

.. describe:: CONFIG_PAGE_TABLE_ISOLATION=y

   Enable KPTI to prevent Meltdown attacks and, more generally, reduce the
   number of hardware side channels.

.. ---

.. describe:: CONFIG_INTEL_TXT=n

   CLIP OS does not use Intel Trusted Execution Technology.

.. ---

.. describe:: CONFIG_HARDENED_USERCOPY=y

   Harden data copies between kernel and user spaces, preventing classes of
   heap overflow exploits and information leaks.

.. describe:: CONFIG_HARDENED_USERCOPY_FALLBACK=n

   Use strict whitelisting mode, i.e., do not ``WARN()``.

.. describe:: CONFIG_FORTIFY_SOURCE=y

   Leverage compiler to detect buffer overflows.

.. describe:: CONFIG_FORTIFY_SOURCE_STRICT_STRING=n

   This extends ``FORTIFY_SOURCE`` to intra-object overflow checking. It is
   useful to find bugs but not recommended for a production kernel yet.
   [linux-hardened]_

.. describe:: CONFIG_STATIC_USERMODEHELPER=n

   This feature requires userspace support that is not currently present in
   CLIP OS.

.. ---

.. describe:: CONFIG_SECURITY=y

   Enable us to choose different security modules.

.. describe:: CONFIG_SECURITY_SELINUX=y

   CLIP OS intends to leverage SELinux in its security model.

.. describe:: CONFIG_SECURITY_SELINUX_BOOTPARAM=n

   We do not need SELinux to be disableable.

.. describe:: CONFIG_SECURITY_SELINUX_DISABLE=n

   We do not want SELinux to be disabled. In addition, this would prevent LSM
   structures such as security hooks from being marked as read-only.

.. describe:: CONFIG_SECURITY_SELINUX_DEVELOP=y

   For now, but will eventually be ``n``.

.. ---

.. describe:: DEFAULT_SECURITY_DAC=y

   The default security module will be changed to SELinux once CLIP OS fully
   uses it.

.. ---

.. describe:: CONFIG_SECURITY_YAMA=y

   The Yama LSM currently provides ptrace scope restriction (which might be
   redundant with CLIP-LSM in the future).

.. ---

.. describe:: CONFIG_INTEGRITY=n

   The integrity subsystem provides several components, the security benefits
   of which are already enforced by CLIP OS (e.g., read-only mounts for all
   parts of the system containing executable programs).

.. ---

.. describe:: CONFIG_SECURITY_PERF_EVENTS_RESTRICT=y

   See documentation about the ``kernel.perf_event_paranoid`` sysctl below.
   [linux-hardened]_

.. ---

.. describe:: CONFIG_PAGE_SANITIZE=y

   Zero-fill page allocations on free to reduce risks of information leaks and
   help mitigate a subset of use-after-free vulnerabilities. This is a simpler
   equivalent to upstream's ``CONFIG_PAGE_POISONING_ZERO``. [linux-hardened]_

.. describe:: CONFIG_PAGE_SANITIZE_VERIFY=y

   Verify that newly allocated pages are zeroed to detect write-after-free
   bugs. [linux-hardened]_

.. ---

.. describe:: CONFIG_SECURITY_TIOCSTI_RESTRICT=y

   This prevents unprivileged users from using the TIOCSTI ioctl to inject
   commands into other processes which share a tty session. [linux-hardened]_

We incorporated most of the *Lockdown* patch series into the CLIP OS kernel,
though it may be merged into the mainline kernel in the near future.
Basically, *Lockdown* tries to disable many mechanisms that could allow the
superuser to eventually run untrusted code in kernel mode (note that a
significant portion of them are already disabled in the CLIP OS kernel due to
our custom configuration). This is an interesting work for CLIP OS as we want
to avoid persistence on a compromised machine even in the case of an
already-root attacker. Among the several configuration options brought by
*Lockdown*, we enable the following ones:

  .. describe:: CONFIG_LOCK_DOWN_KERNEL=y
                CONFIG_LOCK_DOWN_MANDATORY=y

Similarly, we incorporated the *STACKLEAK* feature ported from grsecurity/PaX
by Alexander Popov and which should be merged upstream ultimately. *STACKLEAK*
erases the kernel stack before returning from system calls in order to reduce
the information which kernel stack leak bugs can reveal. It also blocks kernel
stack depth overflows caused by ``alloca()``, such as Stack Clash attacks.

  .. describe:: CONFIG_GCC_PLUGIN_STACKLEAK=y
                CONFIG_STACKLEAK_TRACK_MIN_SIZE=100
                CONFIG_STACKLEAK_METRICS=n
                CONFIG_STACKLEAK_RUNTIME_DISABLE=n


Compilation
-----------

GCC version 7.3.0 or higher is required to fully benefit from retpolines
(``-mindirect-branch=thunk-extern``).


Sysctl Security Tuning
----------------------

Many sysctls are not security-relevant or only play a role if some kernel
configuration options are enabled/disabled. In other words, the following is
tightly related to the CLIP OS kernel configuration detailed above.

.. describe:: kernel.kptr_restrict = 2

   Hide kernel addresses in ``/proc`` and other interfaces, even to privileged
   users.

.. describe:: kernel.yama.ptrace_scope = 1

   Enable the ptrace scope restriction provided by the Yama LSM.

.. describe:: kernel.perf_event_paranoid = 3

   This completely disallows unprivileged access to the ``perf_event_open()``
   system call. Note that this requires a patch included in linux-hardened (see
   `here <https://lwn.net/Articles/696216/>`_ for the reason why it is not
   upstream), otherwise it is the same as setting this sysctl to ``2``. This is
   actually not needed as we already enable
   ``CONFIG_SECURITY_PERF_EVENTS_RESTRICT``.

.. describe:: kernel.tiocsti_restrict = 1

   This is already forced by the ``CONFIG_SECURITY_TIOCSTI_RESTRICT`` kernel
   configuration option that we enable.

The following two sysctls help mitigating TOCTOU vulnerabilities by preventing
users from creating symbolic or hard links to files they do not own or have
read/write access to:

  .. describe:: fs.protected_symlinks = 1
                fs.protected_hardlinks = 1

In addition, the following other two sysctls impose restrictions on the
opening of FIFOs and regular files in order to make similar spoofing attacks
harder:

  .. describe:: fs.protected_fifos = 2
                fs.protected_regular = 2

We do not simply disable the BPF Just in Time compiler as CLIP OS plans on
using it:

  .. describe:: kernel.unprivileged_bpf_disabled = 1

     Prevent unprivileged users from using BPF.

  .. describe:: net.core.bpf_jit_harden = 2

     Trades off performance but helps mitigate JIT spraying.

.. describe:: kernel.deny_new_usb = 0

   The management of USB devices is handled at a higher level by CLIP OS.
   [linux-hardened]_

.. describe:: kernel.device_sidechannel_restrict = 1

   Restrict device timing side channels. [linux-hardened]_

.. describe:: fs.suid_dumpable = 0

   Do not create core dumps of setuid executables.  Note that we already
   disable all core dumps by setting ``CONFIG_COREDUMP=n``.

.. describe:: kernel.pid_max = 65536

   Increase the space for PID values.

For each hardware configuration, a predefined fixed list of modules for
hardware support will be loaded in the kernel at boot time. Once all modules
are loaded and the system is considered booted, module loading will be disabled
by setting ``kernel.modules_disabled = 1``.

Pure network sysctls (``net.ipv4.*`` and ``net.ipv6.*``) will be detailed in a
separate place.


Command line parameters
-----------------------

We pass the following command line parameters to the kernel:

.. describe:: extra_latent_entropy

   This parameter provided by a linux-hardened patch (based on the PaX
   implementation) enables a very simple form of latent entropy extracted
   during system start-up and added to the entropy obtained with
   ``GCC_PLUGIN_LATENT_ENTROPY``.

.. describe:: pti=on

   This force-enables KPTI even on CPUs claiming to be safe from Meltdown.

.. describe:: spectre_v2=on

   Same reasoning as above but for the Spectre v2 vulnerability.

.. describe:: spec_store_bypass_disable=seccomp

   Same reasoning as above but for the Spectre v4 vulnerability. Note that this
   mitigation requires updated microcode for Intel processors.

.. describe:: iommu=force

   Even if we correctly enable the IOMMU in the kernel configuration, the
   kernel can still decide for various reasons to not initialize it at boot.
   Therefore, we force it with this parameter.

.. describe:: slub_debug=F

   The ``F`` option adds many sanity checks to various slab operations. Other
   interesting options that we considered but eventually chose to not use are:

    * The ``P`` option, which enables poisoning on slab cache allocations,
      disables the ``SLAB_SANITIZE`` and ``SLAB_SANITIZE_VERIFY`` features from
      linux-hardened. As they respectively poison with zeroes on object freeing
      and check the zeroing on object allocations, we prefer enabling them
      instead of using ``slub_debug=P``.
    * The ``Z`` option enables red zoning, i.e., it adds extra areas around
      slab objects that detect when one is overwritten past its real size.
      This can help detect overflows but we already rely on ``SLAB_CANARY``
      provided by linux-hardened. A canary is much better than a simple red
      zone as it is supposed to be random.

Also, note that:

* ``slub_nomerge`` is not used as we already set
  ``CONFIG_SLAB_MERGE_DEFAULT=n`` in the kernel configuration.
* ``page_poison`` is not needed by the page poisoning implementation provided
  by linux-hardened patches.
* ``l1tf``: The built-in PTE Inversion mitigation is sufficient to mitigate
  the L1TF vulnerability as long as CLIP OS is not used as an hypervisor with
  untrusted guest VMs. If it were to be someday, ``l1tf=full,force`` should be
  used to force-enable VMX unconditional cache flushes and force-disable SMT
  (note that an Intel microcode update is not required for this mitigation to
  work but improves performance by providing a way to invalidate caches with a
  finer granularity).

.. rubric:: Citations and origin of some items

.. [linux-hardened]
   This item is provided by the ``linux-hardened`` patches.

.. vim: set tw=79 ts=2 sts=2 sw=2 et:
