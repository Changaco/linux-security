This document summarizes my deeper research into Linux security which started in April 2026 with a focus on sandboxing.

## Comparison of Linux sandboxing tools

### Sandboxes usable with little or no knowledge of the details

[firejail](https://firejail.wordpress.com/) is the well-established Linux application sandbox, dating back to [April 2014](https://github.com/netblue30/firejail/blob/46a111166e23a2a1c99bb079616c0fd8687ae533/RELNOTES#L1676-L1680). Written in C, firejail makes it easy to put some restrictions on a program, but difficult to only give a program access to the strict minimum it needs to run. I've contributed a [quick-and-dirty fix for a known vulnerability](https://github.com/netblue30/firejail/pull/7154).

[bubblejail](https://github.com/igo95862/bubblejail) was created in November 2019. Written in Python, it's meant to be a firejail alternative with a different technical approach. bubblejail comes with a generic profile and only a small number of application-specific profiles, whereas firejail comes with a default profile and more than a thousand app-specific profiles.

#### Tied to a specific package format

These projects tie sandboxing to a specific way of packaging and distributing applications, which is problematic because other installation methods will keep existing. [The community arguably didn't need such a system, and it definitely didn't need two competing ones.](https://distrowatch.com/weekly.php?issue=20160704#opinion)

[flatpak](https://github.com/flatpak/flatpak) had its first commit in December 2014 and has had its current name [since 2016](https://lists.freedesktop.org/archives/flatpak/2016-May/000204.html). Written in C, it has been describing itself as “the future” [for a decade](https://web.archive.org/web/20160607053606/https://flatpak.org/). [Flatpak's sandboxing](https://docs.flatpak.org/en/latest/sandbox-permissions.html) has weaknesses ([1](https://www.linuxjournal.com/content/when-flatpaks-sandbox-cracks-real-life-security-issues-beyond-ideal), [2](https://www.openwall.com/lists/oss-security/2026/05/19/1)).

[snapd](https://github.com/canonical/snapd) also had its first commit in December 2014. Written in Go, it isn't officially supported by all the major distributions. [Snap's sandboxing](https://snapcraft.io/docs/explanation/security/) is also imperfect.

### Lower-level sandboxes

These tools aren't meant to be used directly by the average person.

[sydbox](https://gitlab.exherbo.org/sydbox/sydbox) has a long history: first commit in February 2009. Its current major version, a rewrite from C to Rust, was created in September 2023. syd is designed to sandbox package builds. The [“bugs” section of syd's manual](https://man.exherbo.org/syd.7.html#BUGS) describes problematic limitations of the Linux kernel.

[systemd](https://en.wikipedia.org/wiki/Systemd)'s initial commit was in November 2009. Its first sandboxing directives were added in [April 2010](https://github.com/systemd/systemd/commit/15ae422b7471cf6f41ccf450243d8afd8ea0a054) and many more were added over the years. Written in C, systemd provides the [`systemd-analyze security`](https://man.archlinux.org/man/systemd-analyze.1#:~:text=systemd-analyze%20security%20[UNIT...]) command to facilitate reviewing and improving how the system services are sandboxed.

[systemd-nspawn](https://man.archlinux.org/man/systemd-nspawn.1) was first committed in [March 2011](https://github.com/systemd/systemd/commit/88213476187cafc86bea2276199891873000588d). Written in C, it's designed to manage containers but can also be used to run a non-containerized program in a sandbox (e.g. `systemd-nspawn -D / --volatile -a -- ls -l /`).

[systemd-run](https://man.archlinux.org/man/systemd-run.1) had its [first commit in 2013](https://github.com/systemd/systemd/commit/c2756a68401102786be343712c0c35acbd73d28d). Written in C, it may allow a program to break out of some sandboxes, as it enables forking the user session manager process. systemd-run is indiscreet, logging the full command line in the systemd journal, which can be undesirable in some cases.

[nsjail](https://nsjail.dev/)'s first commit was in May 2015. Written in C++, it notably supports giving the sandboxed program limited network access without needing admin privileges (e.g. `nsjail -Mo --chroot / --use_pasta -- /usr/bin/curl …`).

[bubblewrap](https://github.com/containers/bubblewrap) had its first commit in February 2016 but was [derived from older projects](https://blogs.gnome.org/alexl/2018/06/20/flatpak-a-history/). Written in C, it notably [doesn't support giving applications Internet access without giving them access to the local network and abstract sockets](https://github.com/containers/bubblewrap/issues?q=is%3Aissue%20state%3Aopen%20network).

[gVisor](https://gvisor.dev/)'s first commit was in April 2018. This big project reimplements large parts of the Linux kernel in Go to minimize the risk of a kernel bug being exploited by a sandboxed program. It's primarily meant to be used on servers, but it might also be useful on end-user devices.

[sandwine](https://github.com/hartwork/sandwine) had its first commit in February 2023. Written in Python, it's specific to running Windows applications with [Wine](https://www.winehq.org/). sandwine is built on bubblewrap, so it currently can't fully sandbox applications that need Internet access.

[landrun](https://github.com/Zouuup/landrun) was first committed in March 2025. Written in Go, it doesn't close all the ways out of the sandbox.

### Sandboxes for development environments

This category of sandboxing tools has arisen in reaction to the supply chain attacks that have targeted developers in the last few years. All the tools below had their first commit in 2025.

[drop](https://github.com/wrr/drop) (July 2025) is written in Go. Its `drop` command is meant to isolate a specific directory. It has to be called explicitly, which leaves the risk of forgetting to do so.

[litterbox](https://github.com/Gerharddc/litterbox) (August 2025) is written in Rust. Unlike the other tools mentioned in this section, it's built on containers. As noted in its README, that doesn't make it safer.

[island](https://github.com/landlock-lsm/island) (October 2025) is also written in Rust. Unlike drop, island is designed to automatically isolate a specific directory, thanks to a `zsh` integration that enables and disables the sandboxing based on the current working directory. Its README warns that it's “not yet ready for production use”.

[sheld](https://github.com/pierrelegall/sheld) is also from October 2025, is also written in Rust, and also provides shell integration, but it's designed to sandbox only specific commands, not all the commands executed in a specific directory. sheld is built on bubblewrap, so it currently can't fully sandbox programs that need Internet access.

Note: shell integrations suffer from [limitations](https://github.com/landlock-lsm/island/blob/7735885d0343562e1daf36012ecfd276ef1129e3/assets/shell/hook.zsh#L54-L64). Those could be fixed by modifying the shells, but I think the better solution would be to [run the shells themselves in a partial sandbox that catches all `execve` syscalls](https://github.com/pierrelegall/sheld/issues/1#issuecomment-4316290849).

### Sandboxes for AI agents

[sandlock](https://github.com/multikernel/sandlock) is the youngest tool (March 2026). Written in Rust, it's designed to limit what AI agents are allowed to do. Its source code was itself generated at least partly with AI. [It uses seccomp notifications in a way that is probably still unsafe.](https://github.com/multikernel/sandlock/issues/27)

## Interactive network filtering

[opensnitch](https://github.com/evilsocket/opensnitch) had its first commit in April 2017. Written in Python, Go and C, it gives the user fined-grained control over the network connections established by all applications.

## Linux Security Modules

The [LSM](https://en.wikipedia.org/wiki/Linux_Security_Modules) framework enables kernel modules to enforce custom access control rules.

[SELinux](https://en.wikipedia.org/wiki/SELinux) was first released in [December 2000](https://marc.info/?l=linux-kernel&m=97749381725894) and merged into the kernel in 2003. It's widely used, including on many end-user devices as part of the [Android Application Sandbox](https://source.android.com/docs/security/app-sandbox).

[AppArmor](https://en.wikipedia.org/wiki/AppArmor) has had its current name since 2005 and was merged into the kernel in 2010. It's notably used in Ubuntu.

[Yama](https://www.kernel.org/doc/html/latest/admin-guide/LSM/Yama.html) was merged into the kernel in 2012. This small module implements further restricting the use of the [ptrace](https://en.wikipedia.org/wiki/Ptrace) system call.

[Landlock](https://docs.kernel.org/userspace-api/landlock.html) was first published in 2016 and merged into the kernel in [May 2021](https://lore.kernel.org/linux-security-module/161992097730.25025.14142306394400248363.pr-tracker-bot@kernel.org/T/). Unlike previous modules designed for system administrators, Landlock is [designed for unprivileged use](https://landlock.io/talks/2024-06-06_landlock-article.pdf). So, it only allows adding restrictions, not removing them, whereas the power to modify SELinux or AppArmor policies is also the power to lift restrictions on any application.

## Future work

### More sections?

This document could be expanded to cover more Linux security topics. Some possible topics can be found in the [“Security” article of the Arch Linux wiki](https://wiki.archlinux.org/title/Security) and in [Kicksecure's hardening](https://www.kicksecure.com/wiki/Comparison_with_Others).

### Wikipedia article?

The information gathered here could be used to start a [“Linux sandboxing” Wikipedia article](https://en.wikipedia.org/wiki/Linux_sandboxing). In addition to listing the available tools, the article could explain the basics of how they work. Related articles that already exist include [Sandbox (computer security)](https://en.wikipedia.org/wiki/Sandbox_\(computer_security\)), [Containerization (computer)](https://en.wikipedia.org/wiki/Containerization_\(computing\)), [seccomp](https://en.wikipedia.org/wiki/Seccomp), [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces), [Firejail](https://en.wikipedia.org/wiki/Firejail), [Flatpak](https://en.wikipedia.org/wiki/Flatpak) and [Snap (software)](https://en.wikipedia.org/wiki/Snap_\(software\)).

### Audits?

I'm not aware of any of the existing sandboxing tools having ever been fully audited.

### Harmonized hardening of systemd units across distributions?

Many system services are still running with more privileges than they need. Running `systemd-analyze security` often shows that most units are rated as `UNSAFE`. (These ratings are solely based on a unit's configuration, they don't take into account that some daemons start with a lot of privileges but quickly drop them.)

[SandboxDB](https://sandboxdb.org/) provides comparisons of how different distributions sandbox the same service, but it hasn't been updated since December 2022. Its [source code](https://github.com/erdnaxe/sandboxdb.org) could be improved to support updating the data continuously and efficiently.

### Kernel changes?

The Linux kernel should almost certainly be modified to enable the creation of better userspace sandboxing tools. The creator of sandlock started working on [adding a new feature to seccomp](https://lore.kernel.org/all/?q=f%3Axiyou.wangcong%40gmail.com+s%3Aseccomp) after I pointed out the race condition vulnerability. I've also suggested [adding a new feature to Landlock](https://github.com/landlock-lsm/linux/issues/61) to facilitate the sandboxing of applications that need Internet access.

### A universal sandboxing tool?

A new sandboxing tool is probably needed, as the existing ones all appear to be too limited in scope. This new tool would have to be designed to solve all the sandboxing needs of unprivileged users, including the protection of an entire session (i.e. ensuring that all programs are executed with appropriate restrictions regardless of how and where they're installed and launched).

## Staying up to date

You can [watch this repository](https://github.com/Changaco/linux-security/subscription) or add its [commits feed](https://github.com/Changaco/linux-security/commits.atom) to your news aggregator.

You might also want to subscribe to the [oss-security mailing list](https://oss-security.openwall.org/wiki/mailing-lists/oss-security) if you haven't already, but you should keep in mind that being subscribed to bad news can degrade your mental health.

## Funding

This research has been implicitly crowdfunded by the [Liberapay](https://liberapay.com/) community. If you've benefited from it, please consider donating to [the Liberapay team](https://liberapay.com/Liberapay) or to [me personally](https://liberapay.com/Changaco).

## Copyright

You may freely share and adapt this work without asking for permission, insofar as you comply with the [CC By-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) license.

There is 0% of AI generated content in this repository.
