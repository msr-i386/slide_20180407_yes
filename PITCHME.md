# yesコマンドは速くなっている

@第35回シェル芸勉強会 大阪サテライト

---
# 目次
* `yes`とは
* 速度指標
* `yes`の比較
* いかにして速くなったか

---
# 自己紹介

* ハンドルネーム: MSR
    * [Twitter ID: @msr386](https://twitter.com/msr386)
* Webブラウザ [Tungsten](https://app.tungsten-start.net/) の作者

---
# `yes`とは

* 自分がkillされるまで指定した文字列を繰り返し出力するコマンド
* 引数を省略した場合は`y`が出力され続ける

---
# 活用例

- 続行するのに'y'とEnterキーを押す必要のあるプログラム/シェルスクリプトで`y`を勝手に押して欲しい場合
- 負荷試験として
- __シェル芸の足がかりとして__

---

適当にディレクトリを作り、その中で tree したら次のような出力が得られるようにしてください。  
(第33回 シェル芸勉強会 Q1)

```
$ tree
.
└── 💩
    └── 💩
        └── 💩
            └── 💩
                └── 💩
                    └── 💩
                        └── 💩
                            └── 💩
                                └── 💩
                                    └── 💩
```

---

解答例

```
$ yes 💩 | head -n 10 | sed -z "s/\n/\//g;s/^/mkdir -p /" | sh
```


---
# 今回の目的

* yesコマンドが高速化されたことを確かめる

---
# 速度指標

## _yes per second_

---

## yes per second

* 1秒間に何回'y'(+改行コード)を出力することができたかを表す
* 計測にはpvコマンド (_Pipe Viewer_) を使用する  
  debian系なら`apt-get install pv`で簡単に入手可能
* ただし、pvコマンドの結果では2バイトのため単位変換が必要  
  例) 100 [MiB / s] => 52,428,800 [yes / s]
* 「yes per second」ではなく「y per second」ではないか説

---
## pvコマンド実行例

```
$ yes | pv > /dev/null
  63GiB 0:00:02 [1.88GiB/s] [ <=>      ]
```

---

## 計測環境

* PC: GPD Pocket
* OS: Ubuntu 16.04 LTS (Windows Subsystem for Linux)

---

## 最初期

* 1979年1月10日にKen Thompsonによって書かれたコード
* 実に単純明快 (K&RスタイルのC言語なので今だと違和感あるかも)

```
main(argc, argv)
char **argv;
{
  for (;;)
    printf("%s\n", argc>1? argv[1]: "y");
}
```

記録: 14,837,350.4 \[yes/s\]

---
## NetBSDでの実装

* これも非常にシンプル

```
int main(int argc, char **argv)
{
	const char *yes;
	yes = (argc > 1) ? argv[1] : "y";

	while(puts(yes) >= 0)
		continue;

	return EXIT_FAILURE;
}
```


記録: 14,942,208 \[yes/s\]



---
## FreeBSDでの実装

* バッファリングで高速化

※主要部分のみ抜粋、一部改行を削除
```
int main(int argc, char **argv)
{
...
...
	while ((ret = write(STDOUT_FILENO, exp + (explen - more), more)) > 0)
		if ((more -= ret) == 0)
			more = explen;

	err(1, "stdout");
}
```

※計測できず


---
## GNU coreutils

* FreeBSDと同様、バッファリングで高速化

※主要部分のみ抜粋
```
int
main (int argc, char **argv)
{
...
...
  /* Repeatedly output the buffer until there is a write error; then fail.  */
  while (full_write (STDOUT_FILENO, buf, bufused) == bufused)
    continue;
  error (0, errno, _("standard output"));
  return EXIT_FAILURE;
}
```

記録: 225,443,840 \[yes/s\] ※8.25

---
## FreeBSD, GNU coreutilsが高速な理由

* 1行ごとに標準出力するのは効率が悪いので、バッファリングしてから標準出力する
* バッファーは1KiB (`BUFSIZ` @ stdio.h)


---
# 参考(1)

* Unixコマンド”yes”についてのちょっとした話  
https://postd.cc/a-little-story-about-the-yes-unix-command/
* How is GNU `yes` so fast?  
https://www.reddit.com/r/unix/comments/6gxduc/how_is_gnu_yes_so_fast/

---
# 参考(2)

* NetBSD - yes.c  
http://cvsweb.netbsd.org/bsdweb.cgi/src/usr.bin/yes/yes.c?rev=1.9&content-type=text/x-cvsweb-markup
* OpenBSD - yes.c  
https://github.com/openbsd/src/blob/master/usr.bin/yes/yes.c
* FreeBSD - yes.c  
https://github.com/freebsd/freebsd/blob/master/usr.bin/yes/yes.c
* GNU coreutils - yes.c  
https://github.com/coreutils/coreutils/blob/master/src/yes.c

---
# おまけ

\#危険シェル芸  
`` $ yes `yes` ``
