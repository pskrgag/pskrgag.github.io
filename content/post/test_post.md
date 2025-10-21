+++
title = "How to fix a bug in the Linux kernel"
description = ""
date = "2022-03-18"
+++

## Intro

Since the only thing I can do is fixing random kernel bugs, I'd like to show
how to understand and fix bugs reported by syzkaller by example.
There are a lot of open bugs, so anyone who is
looking for first contribution should try it! Most of the bugs really common, but in most cases
maintainers do not have much time to fix bugs. It's great chance to step in and join kernel community by
just adding missing validation check or something

## Configuration

To fix a bug in the Linux kernel you should at least download the sources. The official tree locates
[here](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git) and you can download it like
this

```bash
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

Next step is configure `git send-email`. Linux kernel development process is based on plain text email.
No github pull requests or something. Brutal old-school emails in plain text. Add following configuration
into your `~/.gitconfig`. Mine is for gmail accounts.

```
[sendemail]
	from = Name Surname <email@examle.com>
	smtpserver = smtp.gmail.com
	smtpuser = email@examle.com
	smtpencryption = tls
	smtppass = [pass]
	smtpserverport = 587
	confirm = auto
```

If you use 2 factor authentication `smtppass` field should be unique app password.
You can create it on gmail security page.

Great! You have set up all needed environment.

## Example of bug fixing

For this example I took [this](https://syzkaller.appspot.com/bug?id=f2aa6d86061f7465b5bf6e306d91f52b29051730) bug. It's very simple, but it was open for 120 days! \
Let's take a look at the full report.

```
------------[ cut here ]------------
WARNING: CPU: 1 PID: 3603 at drivers/i2c/i2c-core-base.c:2178 __i2c_transfer+0xa14/0x17c0 drivers/i2c/i2c-core-base.c:2178 drivers/i2c/i2c-core-base.c:2178
Modules linked in:
CPU: 1 PID: 3603 Comm: syz-executor029 Not tainted 5.16.0-rc5-syzkaller #0
Hardware name: Google Google Compute Engine/Google Compute Engine, BIOS Google 01/01/2011
RIP: 0010:__i2c_transfer+0xa14/0x17c0 drivers/i2c/i2c-core-base.c:2178 drivers/i2c/i2c-core-base.c:2178
Code: 0f 94 c7 31 ff 44 89 fe e8 e9 ab 9b fb 45 84 ff 0f 84 26 fd ff ff e8 fb a7 9b fb e8 65 3b 24 fb e9 17 fd ff ff e8 ec a7 9b fb <0f> 0b 41 bc ea ff ff ff e9 9e fd ff ff e8 da a7 9b fb 44 89 ee bf
RSP: 0018:ffffc90002aafce8 EFLAGS: 00010293
RAX: 0000000000000000 RBX: 0000000000000010 RCX: 0000000000000000
RDX: ffff88801c0f3a00 RSI: ffffffff85dc09e4 RDI: 0000000000000003
RBP: ffff88814a058b58 R08: 0000000000000000 R09: ffffffff8ff74ac7
R10: ffffffff85dc0008 R11: 0000000000000000 R12: 0000000000000010
R13: 0000000000000000 R14: ffff88814a058b78 R15: 0000000000000000
FS:  0000000000000000(0000) GS:ffff8880b9d00000(0063) knlGS:0000000056cc62c0
CS:  0010 DS: 002b ES: 002b CR0: 0000000080050033
CR2: 0000000020000000 CR3: 00000000773de000 CR4: 00000000003506e0
DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
Call Trace:
 <TASK>
 i2c_transfer+0x1e6/0x3e0 drivers/i2c/i2c-core-base.c:2269 drivers/i2c/i2c-core-base.c:2269
 i2cdev_ioctl_rdwr+0x583/0x6a0 drivers/i2c/i2c-dev.c:297 drivers/i2c/i2c-dev.c:297
 compat_i2cdev_ioctl+0x419/0x4f0 drivers/i2c/i2c-dev.c:561 drivers/i2c/i2c-dev.c:561
 __do_compat_sys_ioctl+0x1c7/0x290 fs/ioctl.c:972 fs/ioctl.c:972
 do_syscall_32_irqs_on arch/x86/entry/common.c:112 [inline]
 do_syscall_32_irqs_on arch/x86/entry/common.c:112 [inline] arch/x86/entry/common.c:178
 __do_fast_syscall_32+0x65/0xf0 arch/x86/entry/common.c:178 arch/x86/entry/common.c:178
 do_fast_syscall_32+0x2f/0x70 arch/x86/entry/common.c:203 arch/x86/entry/common.c:203
 entry_SYSENTER_compat_after_hwframe+0x4d/0x5c
RIP: 0023:0xf7e6e549
Code: 03 74 c0 01 10 05 03 74 b8 01 10 06 03 74 b4 01 10 07 03 74 b0 01 10 08 03 74 d8 01 00 00 00 00 00 51 52 55 89 e5 0f 34 cd 80 <5d> 5a 59 c3 90 90 90 90 8d b4 26 00 00 00 00 8d b4 26 00 00 00 00
RSP: 002b:00000000fffc8b7c EFLAGS: 00000246 ORIG_RAX: 0000000000000036
RAX: ffffffffffffffda RBX: 0000000000000003 RCX: 0000000000000707
RDX: 00000000200003c0 RSI: 00000000fffc8bd0 RDI: 00000000f7f15000
RBP: 0000000000000000 R08: 0000000000000000 R09: 0000000000000000
R10: 0000000000000000 R11: 0000000000000000 R12: 0000000000000000
R13: 0000000000000000 R14: 0000000000000000 R15: 0000000000000000
 </TASK>
----------------
Code disassembly (best guess):
   0:	03 74 c0 01          	add    0x1(%rax,%rax,8),%esi
   4:	10 05 03 74 b8 01    	adc    %al,0x1b87403(%rip)        # 0x1b8740d
   a:	10 06                	adc    %al,(%rsi)
   c:	03 74 b4 01          	add    0x1(%rsp,%rsi,4),%esi
  10:	10 07                	adc    %al,(%rdi)
  12:	03 74 b0 01          	add    0x1(%rax,%rsi,4),%esi
  16:	10 08                	adc    %cl,(%rax)
  18:	03 74 d8 01          	add    0x1(%rax,%rbx,8),%esi
  1c:	00 00                	add    %al,(%rax)
  1e:	00 00                	add    %al,(%rax)
  20:	00 51 52             	add    %dl,0x52(%rcx)
  23:	55                   	push   %rbp
  24:	89 e5                	mov    %esp,%ebp
  26:	0f 34                	sysenter
  28:	cd 80                	int    $0x80
* 2a:	5d                   	pop    %rbp <-- trapping instruction
  2b:	5a                   	pop    %rdx
  2c:	59                   	pop    %rcx
  2d:	c3                   	retq
  2e:	90                   	nop
  2f:	90                   	nop
  30:	90                   	nop
  31:	90                   	nop
  32:	8d b4 26 00 00 00 00 	lea    0x0(%rsi,%riz,1),%esi
  39:	8d b4 26 00 00 00 00 	lea    0x0(%rsi,%riz,1),%esi
----------------
Code disassembly (best guess):
   0:	03 74 c0 01          	add    0x1(%rax,%rax,8),%esi
   4:	10 05 03 74 b8 01    	adc    %al,0x1b87403(%rip)        # 0x1b8740d
   a:	10 06                	adc    %al,(%rsi)
   c:	03 74 b4 01          	add    0x1(%rsp,%rsi,4),%esi
  10:	10 07                	adc    %al,(%rdi)
  12:	03 74 b0 01          	add    0x1(%rax,%rsi,4),%esi
  16:	10 08                	adc    %cl,(%rax)
  18:	03 74 d8 01          	add    0x1(%rax,%rbx,8),%esi
  1c:	00 00                	add    %al,(%rax)
  1e:	00 00                	add    %al,(%rax)
  20:	00 51 52             	add    %dl,0x52(%rcx)
  23:	55                   	push   %rbp
  24:	89 e5                	mov    %esp,%ebp
  26:	0f 34                	sysenter
  28:	cd 80                	int    $0x80
* 2a:	5d                   	pop    %rbp <-- trapping instruction
  2b:	5a                   	pop    %rdx
  2c:	59                   	pop    %rcx
  2d:	c3                   	retq
  2e:	90                   	nop
  2f:	90                   	nop
  30:	90                   	nop
  31:	90                   	nop
  32:	8d b4 26 00 00 00 00 	lea    0x0(%rsi,%riz,1),%esi
  39:	8d b4 26 00 00 00 00 	lea    0x0(%rsi,%riz,1),%esi
```

Report says there is a `WARN_ON` at drivers/i2c/i2c-core-base.c:2178. Let's take a look what kind of
warning it is.

```c
	if (WARN_ON(!msgs || num < 1))
		return -EINVAL;
```

Hm, looks like somebody passed wrong arguments. Where these numbers come from? This question can be
easily answered by looking at stack trace.

```
 i2c_transfer+0x1e6/0x3e0 drivers/i2c/i2c-core-base.c:2269 drivers/i2c/i2c-core-base.c:2269
 i2cdev_ioctl_rdwr+0x583/0x6a0 drivers/i2c/i2c-dev.c:297 drivers/i2c/i2c-dev.c:297
 compat_i2cdev_ioctl+0x419/0x4f0 drivers/i2c/i2c-dev.c:561 drivers/i2c/i2c-dev.c:561
 __do_compat_sys_ioctl+0x1c7/0x290 fs/ioctl.c:972 fs/ioctl.c:972
 do_syscall_32_irqs_on arch/x86/entry/common.c:112 [inline]
 do_syscall_32_irqs_on arch/x86/entry/common.c:112 [inline] arch/x86/entry/common.c:178
 __do_fast_syscall_32+0x65/0xf0 arch/x86/entry/common.c:178 arch/x86/entry/common.c:178
 do_fast_syscall_32+0x2f/0x70 arch/x86/entry/common.c:203 arch/x86/entry/common.c:203
 entry_SYSENTER_compat_after_hwframe+0x4d/0x5c
```

First 6 entries are common syscall kernel path, so we can simply ignore them. Next one is `compat_i2cdev_ioctl`. This function is `ioctl` handler for `/dev/i2c-*` devices. We are not interested
in purpose of this devices. Something related to i2c. What we really interested in is what this function does -- it copies some data from user-space and pass it deeper into i2c stack.

```c
519 static long compat_i2cdev_ioctl(struct file *file, unsigned int cmd, unsigned long arg)                                                 
520 {                                
521         struct i2c_client *client = file->private_data;
522         unsigned long funcs;     
523         switch (cmd) {           
524         case I2C_FUNCS:          
525                 funcs = i2c_get_functionality(client->adapter);
526                 return put_user(funcs, (compat_ulong_t __user *)arg);
527         case I2C_RDWR: {         
528                 struct i2c_rdwr_ioctl_data32 rdwr_arg;
529                 struct i2c_msg32 __user *p;
530                 struct i2c_msg *rdwr_pa;
531                 int i;           
532                                  
533                 if (copy_from_user(&rdwr_arg,
534                                    (struct i2c_rdwr_ioctl_data32 __user *)arg,
535                                    sizeof(rdwr_arg)))
536                         return -EFAULT;
537                                  
538                 if (rdwr_arg.nmsgs > I2C_RDWR_IOCTL_MAX_MSGS)
539                         return -EINVAL;
540                                  
541                 rdwr_pa = kmalloc_array(rdwr_arg.nmsgs, sizeof(struct i2c_msg),
542                                       GFP_KERNEL);
543                 if (!rdwr_pa)    
544                         return -ENOMEM;
545                                  
546                 p = compat_ptr(rdwr_arg.msgs);
547                 for (i = 0; i < rdwr_arg.nmsgs; i++) {
548                         struct i2c_msg32 umsg;
549                         if (copy_from_user(&umsg, p + i, sizeof(umsg))) {
550                                 kfree(rdwr_pa); 
551                                 return -EFAULT;
552                         }        
553                         rdwr_pa[i] = (struct i2c_msg) {
554                                 .addr = umsg.addr,
555                                 .flags = umsg.flags,
556                                 .len = umsg.len,
557                                 .buf = compat_ptr(umsg.buf)
558                         };       
559                 }                
560                                  
561                 return i2cdev_ioctl_rdwr(client, rdwr_arg.nmsgs, rdwr_pa);
562         }                        
563         case I2C_SMBUS: {        
564                 struct i2c_smbus_ioctl_data32   data32;
565                 if (copy_from_user(&data32,
566                                    (void __user *) arg,
567                                    sizeof(data32)))
568                         return -EFAULT;
569                 return i2cdev_ioctl_smbus(client, data32.read_write,
570                                           data32.command,
571                                           data32.size,
572                                           compat_ptr(data32.data));
573         }                        
574         default:                 
575                 return i2cdev_ioctl(file, cmd, arg);
576         }                        
577 }

```

Next stack trace entry is `i2cdev_ioctl_rdwr`. From just looking at the code we can say, that reproducer
passed `I2C_RDWR` as cmd. Take a closer look at the call

```
return i2cdev_ioctl_rdwr(client, rdwr_arg.nmsgs, rdwr_pa);
```

`rdwr_arg.nmsgs` is user controlled value. It's obvious from `copy_from_user` call on line 533. 
Ok, let's go down to stack. What does `i2cdev_ioctl_rdwr`? I won't copy-paste full function, I will cut it
to very simplified form

```c

static noinline int i2cdev_ioctl_rdwr(struct i2c_client *client,
		unsigned nmsgs, struct i2c_msg *msgs)
{
	u8 __user **data_ptrs;
	int i, res;

	data_ptrs = kmalloc_array(nmsgs, sizeof(u8 __user *), GFP_KERNEL);
	if (data_ptrs == NULL) {
		kfree(msgs);
		return -ENOMEM;
	}

	res = 0;
	for (i = 0; i < nmsgs; i++) {
		/* something */
	}


	res = i2c_transfer(client->adapter, msgs, nmsgs); <--- Hm???

	/* something else */
}
```
Do you see a problem? Let's go back to root case of the bug. There is following warning in `i2c_transfer`.

```c
	if (WARN_ON(!msgs || num < 1))
		return -EINVAL;
```

And as we discovered above `msgs` is user-controlled value. Code must validate passed arguments before
passing them down to i2c transfer functions. `compat_i2cdev_ioctl` check only for `nmsgs` max value,
but not for `nmsgs` being zero.

I come up with this solution

```diff
diff --git a/drivers/i2c/i2c-dev.c b/drivers/i2c/i2c-dev.c
index bce0e8bb7852..cf5d049342ea 100644
--- a/drivers/i2c/i2c-dev.c
+++ b/drivers/i2c/i2c-dev.c
@@ -535,6 +535,9 @@ static long compat_i2cdev_ioctl(struct file *file, unsigned int cmd, unsigned lo
                                   sizeof(rdwr_arg)))
                        return -EFAULT;
 
+               if (!rdwr_arg.msgs || rdwr_arg.nmsgs == 0)
+                       return -EINVAL;
+
                if (rdwr_arg.nmsgs > I2C_RDWR_IOCTL_MAX_MSGS)
                        return -EINVAL;
 
```

# How to test your fix

The easiest way is delegate testing to syzbot. Full communication guide can
be found [here](https://github.com/google/syzkaller/blob/15439f1624735bde5ae3f3b66c1b964a980415b3/docs/syzbot.md), but we need only one command.

```
#syz test: <tree> <branch or commit SHA>
```

For upstream bug use

```
#syz test git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git master
```

Just send an email to mail address that can be found on bug page

<img src="/2022-03-19_00-39.png">

And attach your patch to the email (as file or as plain text). Syzbot will build the kernel and
run reproducer for some time. If it won't hit any bugs you will receive `Reported-and-tested-by` tag.

A bit harder way is to use `qemu`. You will need to build the kernel by yourself and it's topic for separate
post.

# How to format send your patch

Commit your changes using

```
$ git add -u
$ git commit -s
```

`-s` is important! This argument automatically adds `Signed-off-by` with your email and name to commit log.
Patches w/o `Signed-off-by` tag will be automatically rejected. It doesn't
matter if change is correct or not. 1st line is the subject of your patch. The format is following:

```
<subsystem name>: [<subsystem part if present>:] <short description of the change>
```

For this example:

```
i2c: validate user data in compat ioctl
```

The bug itself is in i2c core, so there is no need in specifying subsystem part. Subject line should be 
less than 75 characters in length, but it's not strict rule. The main rule is -- subject must be small and
should describe the change. Put blank line after the subject line and describe the change in more details.
Add why this change is needed, what's the root case and how you have fixed it. You can add anything
you feel related to the patch. Put blank line after commit message and before tags. It makes patches
look more nice. Do not forget to add `Reported-by` tag from bug page or `Reported-and-tested-by` received
from syzbot

<img src="/2022-03-19_00-39.png">

Example patch message

```
    i2c: validate user data in compat ioctl	<- Subject line
						<- Blank line before commit message
    Wrong user data may cause warning in i2c_transfer(), ex: zero msgs.
    Userspace should not be able to trigger warnings, so this patch adds
    validation checks for user data in compact ioctl to prevent reported
    warnings
						<- Blank line before tags
    Reported-and-tested-by: syzbot+e417648b303855b91d8a@syzkaller.appspotmail.com
    Fixes: 7d5cb45655f2 ("i2c compat ioctls: move to ->compat_ioctl()")
    Signed-off-by: Pavel Skripkin <paskripkin@gmail.com>
```

FWIW, commit message itself should be formatted to 75 chars per line. Some maintainers might get mad at
you for wrong formatting. The full guide for submitting patches can be found [here](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/process/submitting-patches.rst).

Ok, your patch is formatted in your local tree. How to post it to mailing lists? First of all use

```
$ git format-patch HEAD~
0001-i2c-validate-user-data-in-compat-ioctl.patch
$ 
```

command. It will create formatted patch email which you can send using `git send-email` command.
Do not hurry with sending it. Smart kernel developers created a script for finding common mistakes in
patches. Do not forget to use it _before_ posting a patch. Simply run this command from kernel root
directory

```
$ ./scripts/checkpatch.pl 0001-i2c-validate-user-data-in-compat-ioctl.patch
WARNING: Possible unwrapped commit description (prefer a maximum 75 chars per line)
#11: 
Reported-and-tested-by: syzbot+e417648b303855b91d8a@syzkaller.appspotmail.com

total: 0 errors, 1 warnings, 9 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0001-i2c-validate-user-data-in-compat-ioctl.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

There is one warning. It can be ignored, because tags _must_ be one-liners. So looks like our patch is
ready for submittion. The last thing we need the list of people interested in our change. There is also
a script for that. It's called `get_maintainer.pl`. Example of usage:

```bash
$ ./scripts/get_maintainer.pl 0001-i2c-validate-user-data-in-compat-ioctl.patch 
Wolfram Sang <wsa@kernel.org> (maintainer:I2C SUBSYSTEM)
Al Viro <viro@zeniv.linux.org.uk> (blamed_fixes:1/1=100%)
linux-i2c@vger.kernel.org (open list:I2C SUBSYSTEM)
linux-kernel@vger.kernel.org (open list)
```

Not so many people... And the last step is send the patch. Just put all needed emails in `git send-email`
arguments and press enter. Like this

```bash
$ git send-email \
> --to=wsa@kernel.org \
> --to=viro@zeniv.linux.org.uk \
> --cc=linux-i2c@vger.kernel.org \
> --cc=linux-kernel@vger.kernel.org \
> 0001-i2c-validate-user-data-in-compat-ioctl.patch
```

You can check `https://lore.kernel.org/` to see if patch reached the lists. Mine can be found [here](https://lore.kernel.org/all/20211230224750.15380-1-paskripkin@gmail.com/). If your patch reached the lists then you
just need to sit down and relax. Sometimes maintainers reply within a day, but average waiting time
for a feedback is 7-8 days. Don't hurry, maintainers are very busy people, so no need to rush.


Good luck in bug fixing and thanks for reading!
