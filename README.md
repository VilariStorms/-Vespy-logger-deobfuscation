# Brief notes: Vespy logger deobfuscation and destruction

[Vespy grabber](https://github.com/vesperlol/Vespy-Grabber-v2.0) is a piece of fairly popular open source python malware that steals discord tokens, roblox cookies and other game accounts.
I noticed that I was encountering it almost daily on discord and youtube. As of right now the repository is still up and it's being used mainly to target young children whose accounts are valued highly - in some cases accounts worth hundreds of dollars are stolen and drained.

### Note if you don't want to read all this, when I get the time I will include a script you can just run to debfuscate the decompiled python file and get the webhook instantly

Vespy is far from the first piece of malware of it's kind and I'm sure we'll see many more variations in the future, especially with the rise of ai generated code making malware more accessible than ever.

Now - on to the fun part! Vespy, like most loggers of it's kind, is distributed as a windows executable created using [pyinstaller](https://pyinstaller.org/en/stable/). You can read up on how pyinstaller turns python files into windows executable files if you like but for now it's not necessary to understand to begin deobfuscating the python code.
[extremecoder's excellent script ](https://github.com/extremecoders-re/pyinstxtractor) pyinstxtractor allows unpacking exe files generated with pyinstaller. This isn't enough to return the obfuscated code to something human readable, however. To do that we must decompile the "compiled bytecode" (.pyc) files. (Bytecode Machine Code, Assembly, Compiled/Decompiled/Disassembled, etc, etc yes it **is** confusing.)

I highly recommend [pycdc/pycdas](https://github.com/zrax/pycdc). It's a lovely python bytecode disassembler and decompiler written in c++. Once you have it installed it's as simple as running `pycdc file.pyc`. By default the output is printed stdout, however, you may want to save it as a file or just pipe to less.

<img src="https://cdn.discordapp.com/attachments/1079562074599993435/1085661550129467392/image.png">
This exe was unpacked with pyinstxtractor. As you can see, we can already tell quite a bit about what it might be based on what modules are present

Now that we have unpacked the executable it's on to the actual deobfuscation! I do not know if the obfuscation in vespy was copied from elsewhere or is unique. The methods used to protect the malicious code are nothing new and are easy to undo but it is a little confusing upon first glance, especially if you have only managed to partially decompile the bytecode. More on that later.

The first thing you may notice are the imports if the grabber is vespy then this will be at the top of the obfuscated python file:
```
import os
import base64
import shutil
import requests
import json
import re
import winshell
import platform
import psutil
import subprocess
import win32api
import sys
import ctypes
import getpass
user = getpass.getuser()
from json import loads
from time import sleep
from win32crypt import CryptUnprotectData
from sqlite3 import connect
from Crypto.Cipher import AES
from threading import Thread
from zipfile import ZipFile
from PIL import ImageGrab
from random import randint
from discord_webhook import DiscordWebhook, DiscordEmbed
from winreg import OpenKey, HKEY_CURRENT_USER, EnumValue
```

~~This is because vespy is a skid who does not know how to~~ ..

Ahem..anyways, the next thing you'll notice is this:

```
__Obf__ = 'Simple Obf'
__import__(f'''{chr(98)}{chr(117)}{chr(105)}{chr(108)}{chr(116)}{chr(105)}{chr(110)}{chr(115)}''').exec(__import__(f'''{chr(109)}{chr(97)}{chr(114)}{chr(115)}{chr(104)}{chr(97)}{chr(108)}''').loads(__import__(f'''{chr(112)}{chr(105)}{chr(99)}{chr(107)}{chr(108)}{chr(101)}''').loads(__import__(f'''{chr(122)}{chr(108)}{chr(105)}{chr(98)}''').decompress(__import__(f'''{chr(98)}{chr(97)}{chr(115)}{chr(101)}{chr(54)}{chr(52)}''').b16decode('.....))))

```

I'll explain what this is and how we can see what it does shortly.. this is taking longer to write than I anticipated and I'm hungry so I'll leave it at this and add the rest in couple of hours because there's quite a bit to cover and I want to explain everything in a way that lets almost anyone with a bit of python experience to recognize, and safely deobfuscate & destroy these loggers.

Okay let's start at the beginning. Ignore `__Obf__ = 'Simple Obf'` it doesn't matter. We can see the the script is importing and executing something - but what exactly is it importinging and what are all those numbers!

It's actually rather simple. You can try [printing them to see what they are lol](https://cdn.discordapp.com/attachments/842501199693086750/1085713835324883024/image.png)

Now that we have that out of the way we can try and understand what this actually does.

```py
__import__('builtins').exec(__import__('marshal').loads(__import__('pickle').loads(__import__('zlib').decompress(__import__('base64').b16decode('obfuscated-code-here'))))
```
Lets start.. at the end. Hehe. So we can clearly tell that the code is base16 encoded. So let's undo that with base64.b16decode(). Next it's uncompressed with zlib and then here's where I want to draw your attention to something. Before you rush to run pickle.loads() you should be aware that both marshal and pickle can be used maliciously to execute some nasty shit. I'm not your mother and so I'm not going to waste my time explaining all the risks that come with reverse engineering code we know to be malicious. You should already be using a vm and takng the appropriate precautions. But like I said I'm not your mother so let's continue.

[](https://cdn.discordapp.com/attachments/1073263218568462356/1085729416186970203/image.png)

If you've been following along so far you will know what to do next and so I will move along to disassembling the final result.

Python already comes with dis so that works just fine here https://docs.python.org/3/library/dis.html#

```py
import dis
dis.dis(skiddedcodeUwU)
```

<img align='left' src='https://cdn.discordapp.com/attachments/1073263218568462356/1085737994658533487/image.png'> If you've never seen disassembled python bytecode the end result may look horrifying but thankfully for us, in this case this is as far as we have to go. Browsing through you should see the webhooks pretty easily as well as get a general idea of what the logger does to an infected pc. Now vespy thinks they're smart by including a ton of decoy webhooks. However, it's only a trivial matter to extract all the webhooks and checking them with a quick python script of our own will find the real one in moments.

```
import requests

# Open the file we saved all the webhooks in and read its contents line by line
with open('urls.txt', 'r') as file:
	urls = file.readlines()

# Loop through each URL and send a GET request
for url in urls:
	url = url.strip() # Remove any extra whitespace or newline characters
	response = requests.get(url)

	# Print the URL followed by its response
	print(f'{url}: {response.text}')
```

[Finding the real webhook in seconds](https://cdn.discordapp.com/attachments/1073263218568462356/1085724143674208336/image-26.png)

Okay, you did it!

You have the webhook, but what now?

Well the [discord api docs explains the various things we can do with a discord webhook](https://discord.com/developers/docs/resources/webhook). The best thing we can do is determine the guild ID and report it to discord.

Sadly there is no way to invite ourselves to the server with just a webook but that doesn't mean we can't delete it! To delete a discord webhook is ridiculously simple. `curl -X DELETE https://webhook.cum` will do the trick. You can also spam the webhook and since they are able to ping "@everyone" we can infuriate whoever's webhook it is with thousands of pings. You can also upload files, send images, etc. A simple script will do the trick but this neat little site also works splendidly https://ytzmo.github.io/webhook_spammer/


### Additional notes, caveats etc

- While most will not, some loggers will have additional layers of obfuscation and hidden, possibly dangerous code you may accidentally run when attempting to deobfuscate - exercise common sense!

- You may be required to switch between python versions so be prepared beforehand

- This text is a heavily abbreviated portion from a much more in depth collection of my random notes on python malware.

- I am not responsible for anything you do or any malicious code you may accidentally execute while attempting to reverse engineer any malicious code.
-  Be sure to delete every active webhook you find to prevent others from being logged.


 
