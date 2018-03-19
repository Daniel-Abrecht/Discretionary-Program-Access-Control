# Unix Program Permission (UPP) Proposal

## Problem statement

The unix permission system is very versatile. It allows to model almost any access restriction with a charming simplicity.
There are two common ways they are used:

 1) To model the permissions of an actual user
 2) To restrict a daemons access to the files it needs


Unix permissions therefore simultaneously aim to control access of users and programs.
Both concepts work perfectly on their own, but if you need both at the same time there is a problem.
If a user starts a program, he'll usually do so with his own user. Most of the time, he can't change
his user and shouldn't be able to do so. But this means the program can access all files of the user
and other programs he used. But it may not be such a good idea to allow every program to write to
a users .bashrc and autostart directories, for example. It would be possible to remove some groups
before starting the program, but we would have to remember to do so every time. And even if we could
somehow automatically set the groups depending on which program is started, this would leave the
problem that programs could abuse other programs that have permission to access a file they don't
to gain access to it themselves. The same problem would arise when using sudo to use a specific
program with a specific user. Using a different user for a program to control it's permissions
has yet another downside. If multiple users start such a program as the same user, and allow that user
to access some of their files, the users could use that to open other users files with said program.
It would be necessary to have a different user for each possible user program combination and often
a lot of ACLs to model the permissions appropriately, which isn't really feasible for normal users.

This is the problem the unix program permission proposal tries to address. The goal is to make it easy
to restrict access of different programs and users simultaneously.

## Can this problem be solved with existing technologies?

You may wonder, couldn't we solve this with ACLs? Sadly, ACLs by themselves aren't enough for this.
They can be really useful if you need to grant access to a file to another user, but can't or don't
want to create a group for everyone who is allowed to access it. I think they are a really good concept,
despite of their frequent abuse by freedesktop people in the past to dynamically allow users access to
graphics and sound cards, which is a horrible idea.

Using a different user for each possible user program combination with sudo could be used to address
this problem, but that would lead to a really big number of users, the user would still have to remember
to always switch to the correct user before using a program (setuid and setgid bits won't help here).
On top of that, each of these user program combinations would need an entry in /etc/sudoers,
and the user would often have to add a lot of ACLs to model the permissions correctly. All this would be a
lot of work for the users, it's difficult to get right, it's error prone, and it's a nightmare to
maintain.

But how about using other existing solutions? In linux, we have SeLinux and Apparmor, why not use them?
SeLinux an Apparmor are great ways to increase the security servers and some desktop related things
by further restricting their access. The problem can indeed be solved this way to some degree, but
it is pretty difficult to do so. The aim of these solutions is quiet different from just determining
if a user or program is allowed to access a file or not depending on the file. You specify everything
a program is or is not allowed to do, and it's not limited to just files. This is great for putting
general restrictions for daemons and programs in place, but I think to get this right to fit the needs
of every individual user is too much work for the user, and therefore just unfeasible. I also don't like
that these solutions can in some cases also be used to grant instead of restrict access.

I believe this problem needs a solution that every user can easily understand and apply, and what's
currently there just doesn't quiet cut it.


## Proposed solution, Unix Program Permissions

In a nutshell, I propose to add a program user id (PUID) and program group id (PGID) in addition to the
already existing existing user and group ids. In addition to that, we need seperate users and groups
for programs in addition to the already existing ones for users.

We can mostly just dublicate the current concept with user being in multiple groups, ACLs, etc. Files
will need an additional mode, uid and gid field for program users and groups.

### Determining if a user is allowed to open a file

UPP only ever restricts access further when activated, it never grants new permissions.
When opening a file, the permissions to do so should be determined as follows.
Calculate the read, write, etc. permissions the same way as without UPP, let's
call them the user permissions. Next, we determine the program permissions.
The program permissions can be calculated basically the same way as the user
permissions, we just use the PUIDs instead of the UIDs, the files pmode instead
of the files mode, and so on. We'll treat PUID 0 special and grant all corresponding
program permissions to it (this includes the execute permission). The last step
is to combine the user permission and the program permission and calculate the
effective permissions. The effective permissions are all permissions the user
permissions and program permissions share. Except if our uid is 0, in which
case UPP can be ignored. It doesn't make sense to put fences around root.


Example:
```
Assume we have uid=1 gid=1 groups(1,2) puid=1 guid=1 pgroups(1)
We want to open a file with mode=0610 pmode=0441 uid=1 gid=2 puid=1 pgid=2
Our and the files uid match, we own the file. The file owners user permissions are rw.
One of our groups match, the files group user permissions are x.
We don't have any others permissions.
The combined user permissions are therefore rwx.
The puid and the files puid match, the program owns the file. The file owners program permissions are r.
We are in none of the pgroups, the group program permissions are none.
Our pothers permissions are x,
The combined program permissions are therefore rx.
The effective permissions are therefore rx, because these are the only permissions that the user and program permissions share.
```

There are a few interesting properties of this approach. If our PUID and GUID
is zero and the PUID and the GUID of a file is zero too, it's as if the PUIDs
GUIDs never existed. This means enabling UPP on a root file system and initialising every
files' puid, pgid and pmode with 0 won't change any permissions at all. The same thing
applies if the other component of every files pmode is 7, which essentially
allows to turn UPP on or off for file systems that don't support it by setting
a pumask when mounting them.

### Automatically applying the user and the setpuid bit in the files pmode

This is where we finally solve the problem. When a program execs another program,
and either the setpuid bit is set, or we are puid 0, our puid is set to the files puid.
If either the setpgid bit is set, or we are pgid 0, our pgid is set to the files pgid.

This means, per default, we get the puid and guid of the first program we launch and therefore
restrict what it can access. After that, all programs it starts will have the same puid and guid,
except if one of them sets the setpuid or setpgid bit.

We also don't have to worry about accidentally restricting access when we don't want it. We can give
interpreters like bash puid 0 and guid 0, which means we keep our current puid and guid. After a login,
our interpreter should have puid 0 and guid 0. Executing another interpreter or a tool like cp isn't a
problem either. We can give terminal emulators setpuid and setpgid, but be careful to make sure that they
don't allow to execute a command from a non-interactive shell or from a command line option, that would
defeat the purpose of all this.

Restricting permissions of applications is now really easy. Just pchmod and pchown your files as usual,
set the puid and guid of your application, and you're done. For example, you could set the puser and
pgroup of your video library to the video group, and set vlcs puid and pgid to the video user and group.
More fine grained configurations are possible, you could add a vlc user and group, or combine this with
acls.


### New c APIs/syscalls

A few new functions to set PUIDs, GUIDs etc. are needed. We can mostly just
dublicate syscalls affecting regular unix permissions. If uid 0 had special
permissions to use some syscalls, for the derived syscalls uid 0 and puid 0
should bouth independently have those permissions.

| function | description |
|----------|-------------|
| setpuid/getpuid | similar to setuid/getuid |
| setgid/getgid | similar to setgid/getgid |
| setpgroups/getpgroups | similar to setgroups/getgroups |
| fpchmod/pchmod | similar to fchmod/chmod |

The stat struct needs to be updated as well. To maintain backward compatiblity,
the kernel can just add syscalls like fstat2, and wait for all of userspace to
adopt the change. After that, the old fstat syscall can be removed, even though
it may take a few years until noone uses it anymore.


### Further considerations

I'm not sure how useful the euid, suid and ruid differentiation with puids (epuid, spuid, rpuid) would be, I would recommend to not introduce such different puids,
since that may introduce unexpected problems.

A kernel should offer an option to enable or disable UPP.
When UPP is enabled, a default pmode of 777 would make sense in that case, because it essentially disables the effects of UPP on such mountpoints.
But a default pmode of 770 is just as desirable, because it prevents programs from being able to accessing files while not preventing the user from accessing them.
Since there are multiple as reasonable options, the default pmode of mount points on which UPP wasn't enabled and where no other pmode was specified should be customizeable, including setpuid and setpgid bits.
A costumizable kernel option to change the default puid and pgid of such mountpoints would also be nice, because it would allow to automatically restrict access of
programs started from removable media.

I would like to standardize some mount options right away. If a file system supports UPP, it should be possible to enable it with the option "upp" and to disable it with "noupp".
On a mountpoint where UPP isn't enabled, there should be the options pfmode, pdmode, and pmode available to set the pmode of files, directories, both or special files respectively, including setpuid and setpgid bits.
There should also be a pgid and puid option in that case to set the default pgid and puid.
The pmode, pfmode, pdmode, puid and pgid options should only be accepted if the noupp option was explicitly specified.

When a program requests the puid, pgid and pmode of a file, but the system doesn't support it, the c library shuld just return 0 for puids and pgids, and 0777 for the pmode.

Some programs may need to be extended to support unix permissions. For example, it would be nice if ls and stat could display them, and su could set the puid and pgid.
Of course, in order to avoid any breakage on systems where UPP isn't used, let's just not display them on such systems if not explicitly requested.
An easy way to only display them if necessary and actually used in case of a file or process would be to check if the puid and pgid are 0.
An other possibility would be to just add an option, but it would be nice to have one that does the other possible solution too, since looking at a large list of things
that have no effect can be annoying and wast a lot of time. 

To manage pusers and pgroups, a libc with "Name Service Switch" support should add support for ppasswd and pgroup fields in /etc/nsswitch.conf
