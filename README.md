# kube-capsh
runs capsh on a pod

![demo](demo.gif)

# Installation
```bash
curl -LO https://github.com/dcasati/kubectl-capsh/raw/main/kubectl-capsh
chmod +x ./kubectl-capsh
sudo mv ./kubectl-capsh /usr/local/bin/kubectl-capsh
```

kubectl-capsh needs `capsh` to run, so when executed for the first time it will download
that file from this repository and install it under `/usr/local/bin/`. If you need to 
manualy fetch and install `capsh` you can grab a copy of it from this repo and move it 
to `/usr/local/bin`.

```bash
CAPSH_STATIC_BIN=libcap2-2.25-capsh-STATIC
curl -LO https://github.com/dcasati/kubectl-capsh/raw/main/bin/${CAPSH_STATIC_BIN}
chmod +x ${CAPSH_STATIC_BIN}
sudo mv ${CAPSH_STATIC_BIN} ${CAPSH_STATIC_BIN_PATH}}/${CAPSH_STATIC_BIN}
```

# Usage

```bash
kubectl capsh <pod>
```
# Details

## Why this?
TL;DR - kubectl-capsh will automate the following laborious process.

You might need to check what `capabilities(7)` a running pod has for security, 
audit or development purposes. You could manualy `exec` into the pod and check for this 
information on `/proc/pid/status`

For example:

```bash
kubectl exec -n kube-system  -it kube-proxy-zdkdn -- grep Cap /proc/1/status
CapInh:	0000003fffffffff
CapPrm:	0000003fffffffff
CapEff:	0000003fffffffff
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
```
then run `capsh` to decode on a Linux host where that util is installed

```bash
$ capsh --decode=0000003fffffffff
0x0000003fffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read
```

You can then check if those capabilities pose a security risk. On Linux, `man capabilities` will give you the information about the capabilities. This is also available online [here](https://man7.org/linux/man-pages/man7/capabilities.7.html)

## How was the capsh compiled?
`capsh` was statically compiled to make it portable - easier to move into a pod.

The following commands where executed on an Ubuntu 20.04 machine:

1. fetch the source code of libcap2-bin

	```bash
	apt-get source libcap2-bin
	```

	Example ouput:

	```bash
	$ apt-get source libcap2-bin
	Reading package lists... Done
	Picking 'libcap2' as source package instead of 'libcap2-bin'
	NOTICE: 'libcap2' packaging is maintained in the 'Git' version control system at:
	https://anonscm.debian.org/git/collab-maint/libcap2.git
	Please use:
	git clone https://anonscm.debian.org/git/collab-maint/libcap2.git
	to retrieve the latest (possibly unreleased) updates to the package.
	Need to get 86.8 kB of source archives.
	Get:1 http://archive.ubuntu.com/ubuntu bionic/main libcap2 1:2.25-1.2 (dsc) [2230 B]
	Get:2 http://archive.ubuntu.com/ubuntu bionic/main libcap2 1:2.25-1.2 (tar) [63.7 kB]
	Get:3 http://archive.ubuntu.com/ubuntu bionic/main libcap2 1:2.25-1.2 (diff) [20.9 kB]
	Fetched 86.8 kB in 1s (105 kB/s)  
	dpkg-source: info: extracting libcap2 in libcap2-2.25
	dpkg-source: info: unpacking libcap2_2.25.orig.tar.xz
	dpkg-source: info: unpacking libcap2_2.25-1.2.debian.tar.xz
	dpkg-source: info: applying ldlibs.patch
	dpkg-source: info: applying setcap-error-message.patch
	dpkg-source: info: applying Don-t-hardcode-build-flags.patch
	dpkg-source: info: applying Syntax-fixes-for-man-pages.patch
	dpkg-source: info: applying Hide-private-symbols.patch
	dpkg-source: info: applying Avoid-sys-capability.h-on-build-architecture.patch
	dpkg-source: info: applying Filter-out-PIE-flags-when-building-shared-objects.patch
	dpkg-source: info: applying Spelling-fixes.patch
	```
1. add `--static` to LDFLAGS for the progs (where capsh is located)

	```bash
	$ cd libcap2-2.25/progs
	```
	edit the `Makefile` and comment the lines that defines if the build will be `DYNAMIC`

	```bash
	--- Makefile.orig	2021-04-09 15:40:54.083726100 -0600
	+++ Makefile		2021-04-09 10:10:30.373726100 -0600
	@@ -8,9 +8,9 @@ PROGS=getpcaps capsh getcap setcap
	
	BUILD=$(PROGS)
	
	-ifneq ($(DYNAMIC),yes)
	+#ifneq ($(DYNAMIC),yes)
	LDFLAGS += --static
	-endif
	+#endif
	LDLIBS += -L../libcap -lcap
	```

1. make the libraries and progs

	```bash
	$ cd ../ && make
	```

	The new static files are available under `progs/`. Double check that the new `capsh` 
	is now static with `ldd` and
	`file`

	ldd -

	```bash
	$ ldd progs/capsh
		not a dynamic executable
	$
	```
	file -

	```bash
	file  capsh
	capsh: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 3.2.0, BuildID[sha1]=c13ddb20944fe4a5b1c4c58b105dd1adbcad1fe8, with debug_info, not stripped
	```


