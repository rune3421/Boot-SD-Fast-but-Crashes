# Boot-SD-Fast-but-Crashes
This is the latest, unfinished version. The websocket is transferring at the right speed, but the SD card can't keep up and crashes. Needs debugging or faster hardware

Hello and welcome to another episode of 
Karl Fiddles Around!

In this episode we’ll be focusing on optimizing the SD FileWrite function, because in the last episode that was identified as the bottleneck for the datastream. 

Because this is a known problem with SD cards, I’ve decided to start from scratch on this, building a brand new sketch from the ground up, and increment our way in to find out what the maximum performance is for my setup, and then integrate that to my current project after. 

The first thing is to select the best base example code to start with, and that will depend on what the problem seems to be. 

A little bit of the research I did in the interim showed that the SDMicro I’m working with can write at about 80Hz, while the fastest sample rate in my sensor array works at 200Hz. Clearly, receiving samples faster than they can be written would be a problem, regardless of other issues, and the overruns might be what’s causing the blackouts as any built-in buffer in the stream ends up full and then flushes. 

So, depending on where the buffer is being stored before the SD write, we may need to: 

Use a buffered print function for the SD itself
Chunk the data before the websocket
Chunk the data after the websocket
Try another storage system other than the SD Micro for the duration of the session, and set a session limit
Stream the data through serial to a laptop program, like RStudio

So, first confirming the SD card properties, I get 

init time: 13 ms

Card type: SDHC
sdSpecVer: 6.00
HighSpeedMode: false

Manufacturer ID: 0X3
OEM ID: SD
Product: SK32G
Revision: 8.5
Serial number: 0X387B1275
Manufacturing date: 12/2021

cardSize: 31914.98 MB (MB = 1,000,000 bytes)
flashEraseSize: 128 blocks
eraseSingleBlock: true
dataAfterErase: zeros

OCR: 0XC0FF8000

SD Partition Table
part,boot,bgnCHS[3],type,endCHS[3],start,length
1,0X0,0X82,0X3,0X0,0XC,0XFE,0XFF,0XFF,8192,62325760
2,0X0,0X0,0X0,0X0,0X0,0X0,0X0,0X0,0,0
3,0X0,0X0,0X0,0X0,0X0,0X0,0X0,0X0,0,0
4,0X0,0X0,0X0,0X0,0X0,0X0,0X0,0X0,0,0

Scanning FAT, please wait.

Volume is FAT32
sectorsPerCluster: 64
fatStartSector:    9362
dataStartSector:   24576
clusterCount:      973584
freeClusterCount:  973538

So the SDfat library should work for the card. However, when trying the first option, the bufferedwrite example, the sketch failed to begin, so I’ll move on to the next option, because the documentation boasts some very fast write times. 

Trying to compile and load the LowLatencyLogger revealed some missing pieces as well, probably because of placeholders in the code I need to update for my particular case. When scaling, I’ll have to work those pieces out. So the first thing is to read through the code…which is less human friendly than prior versions…and figure out which pieces to edit to make a running version. That means it’s tutorial time!

And after some watching videos and some stray research on another project, I came back with another idea…what if I just asked ChatGPT what to do? So…it suggested a code to buffer the websocket inputs to write in chunks. Now it’s time to merge that with the code I’ve got so far…and see what happens!

Merging the code was surprisingly simple, all the way up to reconciling the uint8_t array type to the secondary buffer built by ChatGPT. So…I’ll just ask it what to do to fix that ;)
And with some chatting, I realized that I’d called the buffer as a string, and had to change that to uint8_t also, and then define lengths and such for it as well. Once that was done, it was just a matter of copying over the setup and loop sections into the new version, and check the compile. 

Compile worked! So now I can boot up both of the chips and see if the data flows…and it doesn’t quite. I think it has to do with buffers being too small…so let’s check that. Here’s the error that pops:

Guru Meditation Error: Core  1 panic'ed (LoadProhibited). Exception was unhandled.

Core  1 register dump:
PC      : 0x4008dbd0  PS      : 0x00060730  A0      : 0x800dc387  A1      : 0x3ffb2770  
A2      : 0x0000012c  A3      : 0xffffffff  A4      : 0x0000012c  A5      : 0x00000000  
A6      : 0x3ffb8a28  A7      : 0x00000000  A8      : 0x8008d71c  A9      : 0x3ffb2750  
A10     : 0x3ffc63f0  A11     : 0x00000000  A12     : 0x00000000  A13     : 0x00000001  
A14     : 0x00060b20  A15     : 0x00000001  SAR     : 0x0000001b  EXCCAUSE: 0x0000001c  
EXCVADDR: 0x0000016c  LBEG    : 0x400901d0  LEND    : 0x400901de  LCOUNT  : 0x00000001  


Backtrace: 0x4008dbcd:0x3ffb2770 0x400dc384:0x3ffb27b0 0x400dad1d:0x3ffb27e0 0x400dade7:0x3ffb2800 0x400dbd8e:0x3ffb2820




ELF file SHA256: 925de7c68f302fc0

Rebooting...

So now I’m getting pretty deep in the weeds. I think at this point these crashes indicate I’m reaching the limits of my hardware. I could try increasing the size of the buffers…let’s see if that works. It didn’t. And it’s basically because of an interrupt issue. The main thing is, the processor is trying to write to the SD from the buffer at the same time as the websocket is sending another packet to the buffer, so it gets confused on whether it’s writing or reading. I know this because the call to write to SD is after 5 packets, but 7 packets get lined up in the buffer (so says the Serial) before the processor crashes…so it doesn’t have time to write all five packets and clear the buffer before more packets come in. 

I think what I need to do is split the stream into two channels for two buffers, so one can read while the other writes and just have the websocket alternate where it’s sending its packets until the buffer is full. If that doesn’t work, then it’s just an interrupt problem with the processor that might not be solvable without a hardware upgrade. 

And with a few tries to separate into twin buffers, adjust the number of packets for each write, and all, it turns out…there’s an error in the code! Apparently it is able to complete a full cycle, including the initial write, but then when it’s ready to go back and overwrite, it’s refused access to that part of the buffer…so it *should* be a simple rewrite to give the processor permission to overwrite the buffer each time…maybe by clearing it at the end when the packet count is reset?

That didn’t work. Going back for some more research, it turns out the incoming JSON document is already an array, and deserializing it makes an array out of it. Parsing back out to individual variables for printing to Serial was handy to know that the data was coming in…but instead of re-copying it to a new buffer…I’ll just have the JSON build up as a buffer itself and then print to the SD card as a JSON. Right??

So I’ve done some serious chopping of the code, and instead I’m working on appending JSON files to each other to upload to the SD card. Now I can actually see the JSON document in the serial printing…which is good. But now there’s an error in opening the SD card. A quick check of the calls, and some of the dummy text wasn’t filled in on the SD.open call. Fixed that. 

Another error, now that that’s working, is that the data isn’t structured right when it’s going in…and that’s mainly because the JSON documents don’t know to move each packet to the next line. Should be a quick fix, since the lines are writing to SD now. 

But, with the data coming onto the SD, we should do a quick check on the data to see if there’s still any packet loss…since that was the original goal. 

Aand, after some fenaggling and error printing and optimizing, it turns out that the SD card or writer is actually too slow. I ended up, after trying a couple variants to get every packet written to SD, checking with ChatGPT what the read/write speed was coming in from the websocket, and what the read/write speed is for the SD card…and the websocket is too fast for the SD. And chunking doesn’t solve the problem without dumping packets. So…I’ll have to resort to a bad option to make it work. 

It’s time to throttle the sampling rate. 

It’s at this point that another researcher, Van, joined the team, and I simultaneously lost free time to work on it. So, here’s the latest, unfinished code. It’s still producing the core processor crashes, but it posts some debugging lines to Serial on how the processor and memory are doing before each crash. 

Van, if you read this, you could decide to go back to earlier versions that don’t use an SD card writer, because those work. They only post to Serial, which dumps the information once it’s been displayed, but you’ll be able to do other tuning to the system if you do that. There’s also options I haven’t explored to stream data from Serial onto a laptop, using Excel or R Studio or Python that I haven’t explored. 

Please let me know if you need more assistance on this. I’ve pasted the final code for both sides below. 
