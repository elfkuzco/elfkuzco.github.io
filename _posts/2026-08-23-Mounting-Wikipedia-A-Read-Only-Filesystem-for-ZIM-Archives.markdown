---
layout: default
title: "Mounting Wikipedia - A Read-Only FUSE Filesystem for ZIM Archives"
date: 2026-08-23 04:00:00 +0100
author: "Uchechukwu Orji"
header_buttons: '<a href="https://github.com/elfkuzco/zimfs" class="button" target="_blank">View Project on GitHub</a>'
---

In the UNIX world, there is a common saying that "everything is a file". From your terminal to device drivers,
pipes and the kernel itself, this philosophy underpins how the operating system abstracts complexity. But what happens
when you want to create something that _looks_ like a filesystem but doesn't actually live on disk?

This question is answered by <a href="https://en.wikipedia.org/wiki/Filesystem_in_Userspace" target="_blank">
FUSE (Filesystems in User Space)</a>. FUSE is a mechanism that allows you to implement
custom filesystems entirely in userspace, without touching kernel code. Instead of writing kernel modules (which
are fragile, hard to debug and require elevated privileges), you can write a FUSE filesystem in any programming language with
data backed by sources other than disk. This opens up remarkable possiblities. You can create filesystems backed by cloud storage,
databases, SSH servers or even dynamically generated content. You are only limited by what you can imagine backing a filesystem.

<a href="https://kiwix.org/" target="_blank">Kiwix</a> lets you download all of Wikipedia and various websites and browse it offline.
The download is a ZIM file which is an archive that can hold millions of articles, images and pages. It's compact, fast to search but I
wanted to poke at it to see what's inside. So, I built <a href="https://github.com/elfkuzco/zimfs" target="_blank">zimfs</a> - a read-only
FUSE filesystem that mounts a ZIM file as a normal folder tree. You can run `ls` against it, `cat` a page and follow symlinks without needing
extraction of the full archive or a special viewer.

```sh
$ zimfs wikipedia_fr_all_maxi.zim /mnt/wiki
$ cd /mnt/wiki
$ ls
```

The interesting thing about FUSE is that even though it has a lot of functions to implement to fully satisfy the kernel's request, you only need to
implement a handful of them to get a functional filesystem. For my case which is a read-only filesystem, I only need to implement the following
functions (with their UNIX functions in C in brackets)

- open a file (`open`)
- open a directory (`opendir`)
- read a file (`read`)
- read a directory (`readdir`)
- get a file/directory attributes (`getattr`)

To begin, I started reading the <a href="https://wiki.openzim.org/wiki/ZIM_file_format" target="_blank">ZIM format specification</a>
to understand how the bytes are laid out in the archive and came up with a way to represent them as files. This is one of those undertakings
that make you realize the universe in computing is just a contiguous array of bytes. At the top is an 80-byte header that tells you the number
of entries in the archive, how many clusters there are, byte offsets of where the clusters start and where the path names are stored amongst
other useful information. Clusters are the places where the actual contents of a ZIM archive live. They may be uncompressed or compressed with
`zlib`, `bzip2`, `lzma2` or `zstd`. With the help of a few little-endian functions, I read this information into a `Header` struct.

```
// Validate and decode a ZIM header from the first 80 bytes of an
// archive
func parseHeader(buffer []byte) (*Header, error) {
	if len(buffer) < ZIM_HEADER_LENGTH {
		return nil, InvalidZimHeader
	}

	magicNumber := readUint32(buffer, 0)
	if magicNumber != ZIM_MAGIC_NUMBER {
		return nil, InvalidZimHeader
	}

	id, err := uuid.FromBytes(buffer[8:24])
	if err != nil {
		return nil, InvalidZimHeader
	}

	header := &Header{
		MagicNumber:   magicNumber,
		MajorVersion:  readUint16(buffer, 4),
		MinorVersion:  readUint16(buffer, 6),
		Id:            id,
		EntryCount:    readUint32(buffer, 24),
		ClusterCount:  readUint32(buffer, 28),
        //...and the rest
	}
	return header, nil
}
```

With the metadata information and the specification serving as a guide, I was faced with new challenges/questions:

1. ZIM files can be in the orders of tens of gigabytes and holding such content fully in memory would be impractical. What would be the most efficient way
   of traversing/listing the contents of an archive of 15GB?
2. There is no concept of directory in the ZIM archive which is something that is crucial to every filesystem implementation.
   Paths in the ZIM archive only refer to actual pages. For example, you might have a path `example.com/assets/index.html` but there
   would be no `example.com/assets` directory. How do we resolve requests from the kernel to read a directory like `example.com/assets`?
3. How do we represent the different types of content in a ZIM archive to the different types of files on a regular filesystem?
4. How do we reduce the number of lookup calls to find an entry (or a file)? For context, some ZIMs contains over 7+ million entries and searching through
   the archive everytime would be grossly inefficient.
5. How do we avoid repeated decompression of a cluster every time a user wants to view the contents of an entry in the archive?

## 1. mmap And Lazily Reading the Contents of the Archive

To mitigate the first problem, I resorted to mapping the file in memory. An <a href="https://en.wikipedia.org/wiki/Mmap" target="_blank">mmap</a>
implements demand paging because file contents are not immediately read from disk and initially uses no physical RAM at all. The kernel only
loads pages we actually touch and it can evict clean pages whenever it needs the RAM back. For a 10GB+ archive, this can be the difference
between "it works" and an Out of Memory (OOM).

```
	mapped, err := mmap.Map(f, mmap.RDONLY, 0)
	if err != nil {
		log.Fatalf("failed to mmap zim file at path %s: %v\n", zimPath, err)
	}
	defer mapped.Unmap()

```

Given our filesystem is read-only, we do not worry about the numerous difficulties that could arise from using mmap.

## 2. Synthesizing directories out of thin air

Regarding the second problem of there being no directory paths in an archive, I decided to synthesize directories via inference. When the kernel sends
a request to lookup `example.com/assets/index.html`, it will need to do a lookup of `example.com/assets` first to get it's inode number. But `example.com/assets`
doesn't actually exist in the archive. To satisfy this request, I implemented a binary search algorithm over the archive's path list.
Since the paths are sorted in lexicographical order, the binary search finds the first entry whose path is greater than or equal to the requested path.

```
// Return the first index i >= start such that the entry at i sorts
// at or after (namespace, path)
func (zf *ZimFile) lowerBound(start uint32, namespace rune, path string) (uint32, error) {
	low, high := start, zf.EntryCount
	target := Entry{
		Namespace: namespace,
		Path:      path,
	}
	for low < high {
		mid := (low + high) / 2
		entry, err := zf.GetZimEntryAtIndex(mid)
		if err != nil {
			return 0, err
		}
		if entry.Get().Compare(&target) < 0 {
			low = mid + 1
		} else {
			high = mid
		}
	}
	return low, nil
}
```

For example, if the kernel requests for `example.com/assets`, the search will return the first entry whose path starts with prefix - in this case
`example.com/assets/index.html`. I can then infer that `example.com/assets` must represent a directory even though there is no explicit directory for it
in the archive. I synthesize a directory using the index of the first matching child entry. Keeping this number in the directory entry is intentional so
I can start there while fetching the directory's children when the kernel makes a `readdir` request. Also, When the kernel later asks for `example.com/assets/index.html`,
the same algorithm will be called using the start from it's "parent".

```
func (zf *ZimFile) GetZimEntryFromStart(start uint32, namespace rune, path string) (ZimEntry, error) {
	target := Entry{
		Namespace: namespace,
		Path:      path,
	}
	lowerBound, err := zf.getEntryLowerBound(start, namespace, path)
	if err != nil {
		return nil, err
	}
	dirent, err := zf.GetZimEntryAtIndex(lowerBound)
	if err != nil {
		return nil, err
	}
	entry := dirent.Get()
    // ... code truncated
    // If the entry matches exactly, this must be the exact request we got
	if entry.Equal(&target) {
		return dirent, nil
	} else {
		// If entry starts with target prefix, then, target must be a
        // directory.
		if strings.HasPrefix(entry.Path, target.Path+"/") {
			return NewDirectoryEntry(namespace, path, lowerBound), nil
		}
	}
	return nil, EntryDoesNotExist
}
```

## 3. Redirects, symlinks and zombies

With those challenges out of the way, the next step was to come up with a way to represent the different content types in ZIM archive cleanly to
"files" on the system. The different types of entries in a ZIM archive were modeled as follows:

- Content entries were shown as regular files
- Redirect entries became symlinks pointing back to Content entries
- Deprecated entries and LinkTarget entries were hidden from the user becuase they weren't useful even though they were considered in the binary
  search

This was a good general lesson. With FUSE filesytems, we decide what should be shown as a regular file, symlink, char device, block device, etc.

## 4. Inodes and caching

Each time the kernel requests for a file via it's name, we must give it back an <a href="https://en.wikipedia.org/wiki/Inode">inode number</a>. Every file is associated with
an inode number. It is with this inode number that the kernel performs operations against
the file. zimfs allocates inodes to the kernel and tells it to cache it for one year (after all, it's
a read-only filesystem).
Depending on the programming language you are working with FUSE library, you might never
have to deal with inodes and work with only filepaths. For instance, when I developed <a href="https://github.com/elfkuzco/jsonfs" target="_blank">jsonfs</a>, the C `libfuse` library
only sent requests with filenames and not inode numbers.

```

func (fs *ZimFS) setLookupEntry(op *fuseops.LookUpInodeOp, childID fuseops.InodeID, attrs fuseops.InodeAttributes) {
	op.Entry.Child = childID
	op.Entry.Attributes = attrs

	// We don't spontaneously mutate, so the kernel can cache as long as it wants
	// (since it also handles invalidation).
	op.Entry.AttributesExpiration = time.Now().Add(365 * 24 * time.Hour)
	op.Entry.EntryExpiration = op.Entry.AttributesExpiration
}
```

In addition, we maintain a bounded <a href="https://en.wikipedia.org/wiki/Cache_replacement_policies#LRU" target="_blank">Least Recently Used Cache</a> to keep the data of inodes
in memory and avoid performing re-computation when the kernel requests for the attributes
. The cache is bounded at `10K` entries at the time of writing.

## 5. Clusters and Decompression

Reading a page means finding it's cluster, decompressing it and extracting the right blob. The expensive part is decompression but I decided to
implement a bounded LRU cache again for decoded clusters as it would be wasteful to redo the compression each time we read a file. The cache is capped
at 64MB allowing the system to hold multiple decoded clusters in memory. For uncompressed clusters, their bytes are already sitting in the mmap, so we hand
out a slice into this mapping directly without making any copies.

With the decisions made, we implement the necessary options for our filesystem. FUSE gives us a struct of callbacks and we fill the ones we care about:

```
func (fs *ZimFS) LookUpInode(ctx, op) error { ... }
func (fs *ZImFS) GetInodeAttributes(ctx, op) error {...}
func (fs *ZimFS) ReadDir(ctx, op) error { ... }
func (fs *ZimFS) ReadFile(ctx, op) error { ... }
func (fs *ZimFS) ReadSymlink(ctx, op) error { ... }
```

Most of these are thin translations from FUSE's vocabulary to our parser's vocabulary

- `LookUpInode` - binary search for a path
- `GetInodeAttributes` - get details about the entry such as blob size, file type and other attributes useful to `stat`
- `ReadFile` - find the cluster where the entry resides, decompress it and return the blob
- `ReadSymlink` - resolve a redirect entry
- `ReadDir` - stream the sorted span of top-level children of an entry

With these problems solved, the development of zimfs demonstrated a viable path toward
implementing custom filesystems entirely in userspace. By addressing complex challenges
such as navigating non-existent directory structures and managing memory while working
with these massive archives, I have demonstrated the flexibility of userspace filesystems and their potential for applications involving specialized formats.

## Future Work

While the application works reliably and efficiently with archives of over 1GB, I am
yet to test it on archives in the orders of 10GB and provide memory benchmarks on reads.
I assume this will require further refining of the cache logic and a different algorithm
than binary search for locating entries faster. There are also
<a href="https://github.com/elfkuzco/zimfs#limitations" target="_blank">a couple of known limitations and restrictioins</a>
that I have circumvented but those are not essential to the development of the project. Over the coming weeks, I will address those and provide updates in a later post :)

## References

Aside from the links in the article, here are a couple of materials that I consulted and found helpful in the development of the project and writing of this blog post.

- <a href="https://www.kernel.org/doc/html/next/filesystems/fuse.html" target="_blank">FUSE Documentation</a>
- <a href="https://web.archive.org/web/20180216233455/https://www.ibm.com/developerworks/linux/library/l-fuse/" target="_blank">Develop your own filesystem</a>
- <a href="https://www.cs.nmsu.edu/~pfeiffer/fuse-tutorial/" target="_blank">Writing a FUSE Filesystem: A Tutorial</a>. There are so many posts explaining what FUSE is but literature on actually writing one is almost non-existent.
- <a href="https://github.com/elfkuzco/jsonfs" target="_blank">jsonFS: A FUSE filesystem that mounts a JSON file as a virtual directory structure</a>
