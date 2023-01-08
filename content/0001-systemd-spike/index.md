+++
title = "Builtin SD card readers (on Linux) are evil, I think"
description = "Are you having weird CPU spikes on your Linux laptop for seemingly no reason? It might be your SD card reader! No, really."
date = 2022-12-01
updated = 2023-01-07

[taxonomies]
categories = ["tech support"]
tags = ["linux"]
+++

I've been wondering what my first real blog post could ever be. Something technical, a rant, maybe both? I've had my GH pages blog up for a while with a Hello World, and yesterday after solving an issue I've been struggling with for a while on one of my Linux machines, decided it was the time to put some life into the den. This issue in particular is quite unintuitive if you don't know what to look for, so I thought writing about it could help other people with the same problem.

## A little backstory

My sister had an ASUS laptop gathering dust in a corner, because its keyboard was broken, among other things, but overall it was in very good condition, as well as having a pretty decent spec for its price tag at the time - a 3rd generation i5 CPU, 6GB RAM (which I later swapped for a 16GB kit) and - *gasp* -  all USB 3.0 ports. Unbelievable. I always wanted to have an always running home server I could rely on, and RPis wouldn't cut it for my needs, so I asked her if I could keep it, claiming that it was in such a condition that only a trained computer whizkid like myself could ever find some use for it. So I took it apart, laid its components as the Manufacturer made them inside a medium-size Amazon cardboard box and put it on a shelf above my desk in my bedroom. I did that for better ventilation, I guess.

## The problem

It started happening give or take an year after I started using my sister's ex-laptop as a home server. For some unknown reason, my server was suffering from what only could be described as "IO exhaustion". After a while (usually 15 days, maybe less) since its last reboot, it got consistent, periodic CPU spikes on the *init* process (which is basically systemd on systemd-enabled systems). It slowed down to the point that sometimes I could barely SSH into it or even run sudo, and it could only be solved by a full reboot.

On the first couple times I googled it I didn't really find anything relevant for my case. Some suggested looking for CPU-hungry services, which there weren't any. Other suggested something being afoot in `systemd-homed`, or some thingmabob I don't remember in DBus - it was none of that. Shutting down my Windows VM didn't solve it. syncthing was always running in the background and was a CPU hog whenever it was syncing, but it never stayed its welcome, so couldn't be it. It had to be something in systemd itself causing the CPU spikes, because nothing under it in the process tree justified that atrocity. Whatever it was, it was spawning so fast I couldn't catch it.

## The investigation

So yesterday I started having the issue again, after 7 days of joyful computing. Less than usual! So there I go searching for answers once more, this time being a little bit more restrictive with the search terms. I known where the problem is - systemd. I know the symptoms - cpu spikes. So I typed in the almighty search box: `systemd spikes cpu`. Yeah, I thought about spikes first and then remembered that they're specifically CPU-related, the order of the terms shouldn't matter, right? **Except they do**.

See, `systemd cpu spikes` yields a different set of results, of which the [Archlinux forum thread](https://bbs.archlinux.org/viewtopic.php?id=278426) that helped me figure out the issue appears but as a related thread from a top-level result, while with `systemd spikes cpu` it appears a top-level result. This may or may not be the case on your end, because I'm logged into my Google account and my cookie and geolocation data combined with the search terms created these results.

The conclusion we can draw from that thread is that an unmounted SD card drive makes the device manager trigger happy with polling to the point it causes your system to slow down. Now, due to my house undergoing renovations recently, I had to move my server to my dad's office and couldn't easily test if plugging in an SD card did anything. But if the problem is in the drive I/O itself, couldn't I just like... *delete* it? It's Linux, isn't it? Everything is possible!

## The solution

First we need to figure out which block device represents the SD card reader:

```
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 465,8G  0 disk
├─sda1   8:1    0   976M  0 part /efi
├─sda2   8:2    0  11,2G  0 part
└─sda3   8:3    0 453,7G  0 part /
sdb      8:16   0 931,5G  0 disk
├─sdb1   8:17   0    64G  0 part
└─sdb2   8:18   0 867,5G  0 part /data
sdc      8:32   1     0B  0 disk
```

`sda` is the primary drive the server runs on. `sdb` is an always connected external HDD serving as additional storage. But wait, I don't have any other storage plugged in, and `0B` in size seems weird - so `sdc` is the sd card reader. It will only change in size and be mountable when there is an SD card inside. To double check, we can read `/sys/class/block/sdX/device/model` and look for something like "Card Reader" or "Generic Multi-Card".

```
$ cat /sys/class/block/sdc/device/model
RTS5138 Card Reader Controller
```

Bingo. To triple check, let's read the I/O error counter:

```
$ cat /sys/class/block/sdc/device/ioerr_cnt
0x6660
```

26208 errors. Wow. Let's put an end to this card reader's misery by writing `1` to `/sys/class/block/sdX/device/delete`, which is an operation that needs priviledged access:

```
# echo 1 > /sys/class/block/sdc/device/delete
```

And that's it. If you run `lsblk` again, the drive will no longer be there. As far as the kernel is concerned, it never existed. It takes a little while for the system to go back to normal, probably because there are still pending I/O calls to be resolved.

### Making it permanent

Perhaps, like me, you don't really have an use for a card reader. And if the need ever arises, you could always use an external dongle. There are two ways you could make the change permanent:

- Desolder the card reader from the motherboard and yeet it into oblivion
- Create a startup service that removes the card just like we did right now.

Unless you desolder it, the card reader is going to be initialized again the next time you reboot, and you might forget to delete the drive. That approach is somewhat extreme, and even if I had the server on my bedroom I'd rather not risk losing what has become quite an essential part of my daily workflow.

So let's create a systemd startup service instead. To make sure we'll always be deleting the correct device, since the only realiable assumption is that the primary drive is `sda`, we're going to use the device's physical location instead of its block node, which is [not 100% reliable either](https://utcc.utoronto.ca/~cks/space/blog/linux/PCINamesNotStable), but should be enough for us. Run this on a terminal:

```sh
for loc in /sys/class/scsi_device/*; do
    if cat "$loc/device/model" | grep -Eqi "(card reader|multi-card)"; then
        echo "$(basename "$loc")";
    fi
done
```

It will return something like `X:X:X:X`, which for me was `7:0:0:0`. It goes without saying that you're going to have to reboot to get the reader back if you already removed it, otherwise this won't return anything.

*Update 2023-01-07*: So, this week I finally got the server back in my bedroom, and up until now I hadn't enabled the startup service yet because I wanted to see how it would look like after a reboot. Weirdly, `/sys/class/block/sdc/device/model` no longer reports `RTS5138 Card Reader Controller`, but `Generic Multi-Card` instead. Did me removing it along with whatever was causing its I/O errors somehow erase the OEM info? The hardware location still is `7:0:0:0` though, so it's not that big of deal.

Now we create a file at `/etc/systemd/system/yeet-card-reader.service` (or whatever name you prefer) with the following contents:

```ini
[Unit]
Description=Yeet the card reader
After=local-fs.target
Wants=local-fs.target

[Service]
ExecStart=/usr/local/bin/yeet-card-reader
Type=oneshot

[Install]
WantedBy=multi-user.target
```

That's the service unit, but we also need to create the script that's actually going to the the job. Create another file at `/usr/local/bin/yeet-card-reader` with the following contents:

```sh
#!/bin/bash

loc="7:0:0:0" # change this to your reader's location
dev_path="/sys/class/scsi_device/$loc/device"

if [[ ! -d "$dev_path" ]]; then
    echo "error: Couldn't find device at $loc" >&2
    exit 1
fi

name="$(cat $dev_path/model)"

# sanity check
if ! echo "$name" | grep -Eqi "(card reader|multi-card)"; then
    echo "error: Device is not a card reader" >&2
    exit 1
fi

echo 1 > "$dev_path/delete"
echo "Removed $name"
```

Remember to mark it as executable. Finally, we reload systemd and enable the service:

```
# systemctl daemon-reload
# systemctl enable yeet-card-reader
```

Done! Now you can relax knowing that (hopefully) you never get weird CPU spikes again. And if you do, it might be something more reasonable like a badly optimized program.

## Conclusion

I'm not sure if this is something that happens with every card reader, or even other forms of storage with removable media. Could this happen with DVD drives, for example? Or (external) USB card reader dongles? Does this also happen on other OSes? I don't know, but I hope this post helped. You can [open an issue](https://github.com/manokara/manokara.github.io/issues) at the blog's repo if you have any questions or remarks. See ya!

