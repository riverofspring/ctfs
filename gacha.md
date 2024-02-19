# misc/gacha

Since I actually main Zhongli, I felt obligated to solve this challenge.
![zhongli](assets/zhongli.png)
<sub><sup>(No way, Zhongli spotted outside Kerckhoff Hall??)</sup></sub>

Here's the challenge description:
![chall](assets/gachadesc.png)

We're given the two files ``package.sh`` and ``chall.zip``. Unzipping ``chall.zip`` gives three images:
![fischl](assets/chall/fiscl.png)
![owo](assets/chall/owo.png)
![uwu](assets/chall/uwu.png)

So ``uwu.png`` and ``owo.png`` are literally useless to us as of right now, but let's take a look at the other file.

``package.sh`` contains the following:
```shell
#!/bin/sh
dd if=/dev/urandom ibs=1 count=128 > secret.key
rm -rf chall
mkdir -p chall
cp img/fiscl.png chall/

# add flag to uwu
magick img/uwu.png \
    -weight 50000 -fill red -pointsize 96 \
    -draw "text 50,540 '`cat flag.txt`'" \
    PNG24:flag.png

magick img/owo.png -encipher secret.key chall/owo.png
magick flag.png -encipher secret.key chall/uwu.png
rm flag.png

rm -f chall.zip
zip -9r chall.zip chall/
```
Let's walk through this bit by bit. The first line creates a file called ``secret.key`` that's just a large random number. Then, the directory ``chall`` is removed and re-created in the event that there was anything there to begin with, such that it is now an empty directory. Next, ``fiscl.png`` is copied into ``chall``. This is where stuff gets good.

Searching for ``magick`` returns ImageMagick, a command-line image editor, which we can install quickly. Then, we can analyze the remaining code. It appears to be taking ``img/uwu.png`` (note that this is not the same file as ``chall/uwu.png``) and drawing text over it, notably, it ``cat``s out ``flag.txt``, presumably onto the image. Therefore, we just need to find what ``flag.png`` looks like, and we'll be done.

With that goal in mind, we look through the next commands. ImageMagick provides an ``-encipher`` option, and the documentation says that it uses the AES cipher in Counter mode. It also has a corresponding ``-decipher`` option, which looks like something we'll have to call. The two enciphering lines take ``img/owo.png`` and ``flag.png``, and encipher them with the exact same passphrase to get ``chall/owo.png`` and ``chall/uwu.png``. Finally, we delete ``flag.png`` and zip up ``chall``.

Simply using the images as passphrases for ``-encipher`` doesn't work: they contain vastly different data from ``secret.key`` and all that you get is something like this:
![lol](assets/chall/what.png)

So I stared at this challenge for two whole hours because I kept trying different combinations of images and passphrases for ``-encipher`` and ``-decipher``. I got nowhere sadg.

