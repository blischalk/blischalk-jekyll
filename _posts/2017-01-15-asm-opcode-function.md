---
layout: post
title: "Opcode Script"
description: "Opcodes from Assembly Instructions"
tags: [asm, opcodes,]
---

While working through some of the [corelan.be](https://www.corelan.be)
tutorials I came across the "assemble" functionality in windbg which
allows you to lookup the opcodes of assembly instructions. I have
previously used an online assembler for this sort of functionality
before located at
[defuse.ca](https://defuse.ca/online-x86-assembler.htm) but I wanted
the ability to lookup the instructions on the command line without
going to the web. I looked into the functionality of `gdb` and `nasm`
but it doesn't appear that either provide this sort of functionality
so...

Here is a little shell function I put in my `.bashrc`:

{% highlight bash %}
function asmop {
DIR=safetydir123
mkdir $DIR
cd $DIR
FILE=tmpasm
while read x ; do
  touch $FILE.nasm
cat << EOF > $FILE.nasm
global _start
section .text
_start:
EOF
  echo $x >> $FILE.nasm;
  nasm -f elf32 -o $FILE.o $FILE.nasm
  ld -o $FILE $FILE.o
  objdump -D $FILE -M intel
  rm $FILE*
done
cd ../
rmdir $DIR
}
{% endhighlight %}


This lets us lookup opcodes for assembly like:

{% highlight bash %}
~/$ asmop
jmp esp

tmpasm:     file format elf32-i386


Disassembly of section .text:

08048060 <_start>:
 8048060:	ff e4                	jmp    esp
push eax

tmpasm:     file format elf32-i386


Disassembly of section .text:

08048060 <_start>:
 8048060:	50                   	push   eax
{% endhighlight %}

