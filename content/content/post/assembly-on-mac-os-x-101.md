+++
date = "2016-08-14T18:52:46+08:00"
draft = false
slug = "assembly-on-mac-os-x-101"
tags = ["x64", "C"]
title = "Assembly on Mac OS X 101"
description = "Mac OS X上でのX64アセンブリ言語の実行方法と、C言語とのリンク方法について紹介"
+++
## 概要
- アセンブリ言語は、[x86-64](https://ja.wikipedia.org/wiki/X64)を使用する。
- アセンブラは、[NASM(Netwide Assembler)](http://www.nasm.us/)を使用する。
- OS X上での「Hello World」と「C言語からの関数呼び出し」のサンプルコードを紹介する。

## はじめに

アセンブリ言語は、命令セットアーキテクチャ(x86、ARMなど)やアセンブラ(NASM、[GAS](https://ja.wikipedia.org/wiki/GNU%E3%82%A2%E3%82%BB%E3%83%B3%E3%83%96%E3%83%A9))が採用する構文(Intel、AT&T)、bit数に従って、若干異なる[^1]。  
この記事では、64bitのx86-64(x64)のアセンブリ言語を、NASM(Intel構文)を用いてOS Xの実行ファイル形式Mach-Oに変換する。  

この記事のプログラムは、OS X EL Capitan上にて確認している。実行に際して必要になるものはgccとld、NASMだけであり、NASMはbrewを使って簡単にインストールすることができる。

## Hello World
以下に、Hello Worldのプログラムを示す。  

```nasm
; helloworld.asm

GLOBAL start


SECTION .data
str_hello   db  "Hello World", 0x0a ; Output string and \n


SECTION .text
start:
    mov rax, 0x2000004      ; Set system call to write=4.
    mov rdi, 1              ; Set output to stdout.
    mov rsi, str_hello      ; Set output data.
    mov rdx, 13             ; Set output data size.
    syscall                 ; Call system call.
    mov rax, 0x2000001      ; Set system call to exit=1.
    mov rdi, 0              ; Set success value of exit.
    syscall                 ; Call system call.
```

ここでは、アセンブリ言語の詳細は解説しない。詳しは入門サイトなどを参照されたい[^2][^3]。  

このプログラムは、下記の手続きで実行できる。 `-f`は、出力する実行ファイルのフォーマットを指定している。

``` sh
$ nasm -f macho64 helloworld.asm
$ ld -o helloworld helloworld.o
$ ./helloworld
Hello World
```

## C言語からの関数呼び出し
アセンブリ言語で記述した関数を、C言語とリンクして実行する。--
以下に、アセンブリ言語で記述した二乗関数(square)のプログラムを示す。


```nasm
GLOBAL _square

_square:
    push    rbp
    mov     rbp, rsp
    mov     rax, rdi
    imul    rax, rdi
    mov     rsp, rbp
    pop     rbp
    ret
```

次に呼び出す側のCプログラムを示す。
```c
#include <stdio.h>

extern int square(int);

int main(void)
{
    printf("%d\n", square(12));
    return 0;
}
```

これらのプログラム配下の様にして実行する。

```sh
$ nasm -f macho64 square.asm
$ gcc -o main main.c square.o
$ ./main
144
```

### おわりに
この記事では、OS X上でのX86アセンブリ言語の実行方法と、C言語とのリンク方法について紹介した。



[^1]: [Linux のアセンブラー: GAS と NASM を比較する](http://www.ibm.com/developerworks/jp/linux/library/l-gas-nasm.html)
[^2]: [アセンブラ入門](http://www5c.biglobe.ne.jp/~ecb/assembler/assembler00.html)
[^3]: [x64 アセンブリ言語プログラミング](http://homepage1.nifty.com/herumi/prog/x64.html#GCC64)
