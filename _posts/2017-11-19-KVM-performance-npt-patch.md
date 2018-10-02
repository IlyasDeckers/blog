---
type: post
title: Improving KVM performance for Ryzen with the NPT patch
---
## INTRODUCTION

GPU passthrough on Ryzen platform has its issues. The most common one, and the one I struggled with the most was a 10 year old bug. The one we will address today. For those who already tried GPU passthrough on ryzen with KVM, you should have noticed a serious performance drop in your guest vs running your GPU on bare metal. This issue could be adressed by using `npt=0`. This brought along some other issues like not being able to use host-passthrough, the guest only detecting one core, cpu performance loss,... Far from ideal. Last week [Geoffrey and Paolo](https://patchwork.kernel.org/patch/10027525/) managed to patch this bug. In this article we will be applying the patch and compiling it with the latest linux kernel. (4.14-rc8)

## GETTING STARTED

You can use almost any kernel with this patch. I have tested this with kernel 4.10, 4.13 and 4.14-rc8. In this guide, I will be using kernel 4.14-rc8. Kernel 4.14 will be released very shortly as of writing. Linus Torvalds noted that this past week of 4.14 development was fairly quiet and perhaps doesn't need an "RC8" update, but he opted for it anyways. UPDATE: Kernel 4.14 has been released. [Click here](https://ilyasdeckers.be/2017/11/17/how-to-install-kernel-4-14-in-linux-ubuntu/) to learn how to install and update your kernel.

### Getting the kernel

First, we will get the build files for the new kernel.

```sh
git clone git://git.launchpad.net/~ubuntu-kernel-test/ubuntu/+source/linux/+git/mainline-crack v4.14-rc8
```
## Creating and applying the patch

The patch can be found here: https://patchwork.kernel.org/patch/10027525/ or copy and paste the patch to a new file npt-ryzen.patch
```sh
nano npt-ryzen.patch

diff --git a/arch/x86/kvm/svm.c b/arch/x86/kvm/svm.c
index af256b786a70..af09baa3d736 100644
--- a/arch/x86/kvm/svm.c
+++ b/arch/x86/kvm/svm.c
@@ -3626,6 +3626,13 @@  static int svm\_set\_msr(struct kvm\_vcpu \*vcpu, struct msr\_data \*msr)
 	u32 ecx = msr->index;
 	u64 data = msr->data;
 	switch (ecx) {
+	case MSR\_IA32\_CR\_PAT:
+		if (!kvm\_mtrr\_valid(vcpu, MSR\_IA32\_CR\_PAT, data))
+			return 1;
+		vcpu->arch.pat = data;
+		svm->vmcb->save.g\_pat = data;
+		mark\_dirty(svm->vmcb, VMCB\_NPT);
+		break;
 	case MSR\_IA32\_TSC:
 		kvm\_write\_tsc(vcpu, msr);
 		break;
```

Now that we have the patch we can apply it to our kernel.
```
cd v4.14rc8
cp /boot/config-$(uname -r) .config
patch -p1 < npt-ryzen.patch
```

## Compiling the new kernel

If all went well, it is time to compile our new patched kernel. Remember, after compiling and installing the new kernel you will have to reboot. Execute the following commands to start compiling the kernel.
```sh
yes '' | make oldconfig
make clean
make -j 4 deb-pkg LOCALVERSION=-with-npt
```
Note: this process can take up some time.

## Installing the kernel

After the compiling is done you will be left with some .deb files. These are the installation files that we need in order to install the patched kernel.

```sh
sudo dpkg -i linux-headers-4.14.0-rc6-custom\_4.14.0-rc6-custom-1\_amd64.deb linux-image-4.14.0-rc6-custom\_4.14.0-rc6-custom-1\_amd64.deb  linux-libc-dev\_4.14.0-rc6-custom-1\_amd64.deb
```

At this point you are ready to reboot your machine and your system should be ready for GPU passthrough without any performance loss on your KVM guest.
