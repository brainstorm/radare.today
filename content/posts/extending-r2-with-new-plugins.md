+++
date = "2014-11-09T12:00:00+01:00"
draft = false
title = "Extending r2 with new plugins"
slug = "extending-r2-with-new-plugins"
aliases = [
	"extending-r2-with-new-plugins"
]
+++
One of the key features behind r2 is how easily it can be extended with new libraries or plugins. In this blopost, we'll see the steps to add a new plugin in radare2.

Let's say we want to add a new plugin for `r_asm` because we are working with binaries of an architecture not supported by r2. Of course, adding a new plugin for another lib would be mostly the same.

From now to the end of the document, we will name this new plugin as "*myarch*".

Please note that you can keep your plugin outside the radare2 [main tree]( https://github.com/radare/radare2 ), like the ones in [radare2-extras]( https://github.com/radare/radare2-extras ).

The first thing we have to do is give a look to the r_asm header, located at `libr/include/r_asm.h`, and find out which callbacks we need to implement. They will be always defined under a structure named `r_libname_plugin_t` or `RLibnamePlugin`. In the case of `r_asm`, look closely at `RAsmPlugin`:
```
typedef struct r_asm_plugin_t {
	[...]
	int (*init)(void *user);
	int (*fini)(void *user);
	int (*disassemble)(RAsm *a, RAsmOp *op, ut8 *buf, ut64 len);
	int (*assemble)(RAsm *a, RAsmOp *op, const char *buf);
	RAsmModifyCallback modify;
	int (*set_subarch)(RAsm *a, const char *buf);
} RAsmPlugin;
```

We can implement all the callbacks or only those needed for our purposes. In this case we only want to code the *disassemble* function.

Then, we will create a new file named asm_myarch.c under `libr/asm/p/`. The typical skeleton woud be:
```
static int disassemble(RAsm *a, RAsmOp *op, ut8 *buf, ut64 len) {
	/* TODO: Implement disassemble code here,
     * give a look to the other plugins */
}

RAsmPlugin r_asm_plugin_myarch = {
	.name = "MyArch",
	.desc = "disassembly plugin for MyArch",
	.arch = "myarch",
	.bits = (int[]){ 32, 64, 0 }, /* supported wordsizes */
	.init = NULL,
	.fini = NULL,
	.disassemble = &disassemble,
	.modify = NULL,
	.assemble = NULL,
};

#ifndef CORELIB
struct r_lib_struct_t radare_plugin = {
	.type = R_LIB_TYPE_ASM,
	.data = &r_asm_plugin_myarch
};
#endif
```

Be careful with the name convention to avoid collisions with other existing plugins. Don't forget that you have lots of examples inside `libr/*/p/` on how plugins are implemented.

If we need more files for the plugin to work, e.g. a backend, we will create a folder under `libr/asm/arch/` called like the plugin arch (*myarch*) and copy all the files to `libr/asm/arch/myarch/`.

The next step is to add a new "*extern*" in the header file, so we can compile it as static plugin if we want to. In `libr/include/r_asm.h`:
```
[...]
extern RAsmPlugin r_asm_plugin_x86;
extern RAsmPlugin r_asm_plugin_x86_olly;
extern RAsmPlugin r_asm_plugin_x86_nasm;
[...]
extern RAsmPlugin r_asm_plugin_myarch;
```

Obviously, we need to tell the build system how to compile our code, this is done adding a new file called `libr/asm/p/myarch.mk`, which would seem like this:
```
OBJ_MYARCH=asm_myarch.o
# myarch backend
OBJ_MYARCH+=../arch/myarch/udis86/file1.o
OBJ_MYARCH+=../arch/myarch/udis86/file2.o
[...]

STATIC_OBJ+=${OBJ_MYARCH}
TARGET_MYARCH=asm_myarch.${EXT_SO}

ALL_TARGETS+=${TARGET_MYARCH}

${TARGET_MYARCH}: ${OBJ_MYARCH}
	${CC} ${LDFLAGS} ${CFLAGS} -o ${TARGET_MYARCH} ${OBJ_MYARCH}
```

And appending it to the list of compiled plugins in `libr/asm/p/Makefile`:
```
[...]
ARCHS+=myarch.mk
[...]
```

Finally, we only need to define if it must be compiled as static or shared, which is done editing `plugins.def.cfg`, in the source root. In this case, we will compile it as static, so we add a new line following the other asm plugins:
```
STATIC="asm.java
asm.arm
[...]
asm.x86
asm.myarch
[...]
```
Ok, We have just created our first r2 plugin. Now, exec:
```
$ make mrproper && ./configure && make && make install
```
and enjoy! :)
