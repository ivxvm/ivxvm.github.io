+++
title = "Btrfs for daily driving"
date = 2026-10-23
+++

Initially I wanted to name this post "btrfs for desktop use" to create contrast with server scenarios, but I realized it may be confusing because the term "desktop" is sometimes also used in contrast to "laptop", but in this case I actually mean both desktops and laptops and probably some other personal devices. "Daily driving" might sound a bit "too memey", but it feels like a better choice here.

I first encountered btrfs when I used Bazzite - gaming-oriented atomic immutable distro - a heavily customized Fedora Atomic. I didn't think much of it and the only thing I noticed is that this filesystem was used by default on such a "set it and forget it" distro for gamers. Never had any major issues during my year of using that distro other than being annoyed by needing to use things like distrobox and rpm-ostree to get anything that's not covered by just flatpaks and brew.

Later, when I moved to Gentoo I also picked btrfs for my system volume because it was one of recommended and well-supported options and I already had an impression of it being a good modern general-purpose filesystem after seeing it used in Bazzite. At that point I wasn't really aware of properties of btrfs on deeper level, so it wasn't super deliberate. However, I started discovering them very soon.

Because of Gentoo not being atomic itself, it means things like kernel and driver updates, or just global software updates in general, are more risky: if something goes wrong during update process, there is a possibility of system ending up in broken inconsistent state. Different ways to make it safer exist, the one I decided to try is to use snapshots which btrfs supports natively. Because this filesystem uses copy-on-write (COW), it naturally makes snapshots zero-cost: they don't copy anything until you change any of the files at which point new modified copy of the file will be created while the old one will stay in place, still referenced by snapshot. Many tools exist to automate creating snapshots, however I decided to stick to native btrfs cli, because it turned out very easy to use - I simply have to invoke a single command before doing system updates or anything risky:

```sh
btrfs subvolume snapshot / /snapshots/2026_10_16
```

And that's basically it: zero-cost = no copying anything without need = no disk space wasted (and no LLMs, even though I written "no" quite a few times). What's the catch though, why wouldn't anyone just use filesystems like btrfs universally? That's where I started researching more and got surprised by how unstable and experimental it was considered in its early days according to decade old posts and discussions. Most of those early issues, however, were addressed and fixed long time ago and nowadays the most problematic aspect turns out to be fragmentation which only happens during very specific circumstances. This issue is rather fundamental and stems mostly from COW itself and from other similarly fundamental filesystem design choices. So what are those conditions and how to deal with them?

## 1. Big files that are modified often and in random order (DB & VM volumes)

This is the edge case that is well known on the internet, and is one of the first things to find when looking for problematic btrfs scenarios. The root of the problem is that due to COW, btrfs can't perform random writes in place: each write will allocate new data and old block becomes dead space. Over time this leads to massive fragmentation: the file's physical content gets scattered across the disk as many small extents.

I personally stumbled upon this one in my bitmagnet setup: turns out it uses postgres database for bookkeeping and even though I only kept it spinning for a week or two, when checking fragmentation these postgres files were at multiple dozen thousands extents.

Fortunately, there is a solution to this problem: disable COW! Yep, turns out btrfs supports disabling COW for individual files and folders, and in the later case, all new files and folders created within such a folder, will also have COW automatically disabled.

```sh
# disable COW
chattr +C ~/bitmagnet/data
# check flags (big letter 'C' means COW is disabled)
lsattr ~/bitmagnet/data
```

## 2. Big files that are initialized in small chunks in random order (torrented media)

This one is more tricky, initially I expected +C to solve all problems with COW-related fragmentation, but this is not the case unfortunately: when checking my media drive for files with heavy fragmentation, to my surprise the worst cases were all torrented videos. The problem with those is that in many cases torrent clients by default work with sparse files, and when new data arrives in random order those arrivals are not overwrites to existing data, instead it's new allocations which create fragmented extents. In some torrent clients it's possible to configure them to preallocate the whole file beforehands, which will fix the problem when combined with +C. Otherwise, the solution would be to defragment torrented files manually as a periodic maintenance.

```sh
# defragment file
btrfs filesystem defrag -v -t 32M file
# check file fragmentation
filefrag file
```

## Snapshots and +C

Another important concern is that in case of snapshots, copy-on-write will still happen on the first write to files that are referenced by snapshots even if they have +C flag. So if your DB or VM files live on btrfs volume that you are snapshotting, you might lose quite some disk space after every new snapshot (on the next write to corresponding DB or VM files). The best solution to this I am aware of is to create separate subvolumes for those DB/VM files, because when snapshotting a volume, btrfs will simply ignore any subvolumes and not include them into snapshots. It's not as annoying as it sounds because btrfs makes it easy to create arbitrary subvolumes at arbitrary paths - so that you can have for example `~/bitmagnet/data` as a separate subvolume, while `/` is the main volume that you take snapshots of.

```sh
btrfs subvolume create ~/bitmagnet/data
chattr +C ~/bitmagnet/data
```
## SSD vs HDD

I use btrfs both for my system SSD and external HDD, and there are some important differences. Because SSDs are very fast, they are more forgiving to fragmentation, but because they also wear off, they benefit from minimizing writes by using compression. This is why I ignore some mild cases of fragmentation on SSD - those where files are not frequently written - and use it with zstd compression:

```sh
UUID=...   /   btrfs   rw,relatime,ssd,compress=zstd:3,space_cache=v2,subvol=/ 0 0
```

On my external HDD there is a lot of space and it's significantly slower than SSD. Btrfs compression also leads to some additional fragmentation in certain scenarios, so using compression would be a bad choice for HDD:

```sh
UUID=...  /mnt/expansion  btrfs  defaults,noatime  0  0
```

`noatime` flag prevents rewriting metadata on file access which improves performance in some cases and it might be useful on HDDs. On my main system SSD I decided to leave the default `relatime` in place because some sources say certain system software may need proper access times.
## Impressions so far

It really does feel great to be able to instantly create snapshots that take zero extra disk space (until later). But it's definitely not without a price, and in this case the price is maintenance. I don't see btrfs as a filesystem you can just use and never think about it, in my opinion you need to pay attention: at the very least check fragmentation periodically, move specific types of files into subvolumes and set certain flags when needed. But when used properly, it does have many nice properties, and snapshotting is only one of them.

## Further reading

This blogpost barely scratches the surface, but I believe those aspects I covered are the most important ones for everyday btrfs use. My top 3 resources to find information on btrfs would be:
* [https://btrfs.readthedocs.io/en/latest/](https://btrfs.readthedocs.io/en/latest/)
* [https://lore.kernel.org/linux-btrfs/](https://lore.kernel.org/linux-btrfs/)
* [https://www.reddit.com/r/btrfs/](https://www.reddit.com/r/btrfs/)

## Useful scripts

```sh
# print fragmentation for all files in folder (descending into subfolders recursively)
find /mnt/expansion/videos -type f -exec filefrag {} +
```

```sh
# find files larger than 10M that have >= 1000 fragmentation extents
sudo find / -xdev -type f -size +10M -exec filefrag {} + |
	sed -nE 's/(.+): ([0-9]+).*/\2 \1/p' |
	awk '$1 >= 1000' |
	sort -nr
```
