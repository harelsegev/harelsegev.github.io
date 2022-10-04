---
title: "Resolving File Paths Using the MFT"
date: "2022-07-07"
author: "Harel Segev"
tags: ["NTFS", "MFT"]
draft: false
---

In NTFS, the MFT (Master File Table) is a structure that contains a lot of the file-system metadata, and also the contents of small files. It is stored in a special file, called `$MFT`. In incident response, we often collect and parse this file to determine the file system contents and how it changed over time, without having to acquire a full disk image.

There are many bad MFT parsers out there. This is no coincidence - an MFT parser is an extremely easy thing to mess up. Most MFT parsers don't bother handling the uncommon edge cases - and they work fine, most of the times. The fact is, some of those uncommon edge cases are actually more common than you would think. Handling them correctly is what sets the good MFT parsers apart from the bad ones.

The MFT is an array of  file records - each one describes a file on the system. File records do not to store directory hierarchy, though. There's another structure for that - directory indexes. The directory index a file is listed in determines its location in the directory tree. The directory index is the one storing the name of the file, inside a structure called a `$FILE_NAME` attribute.

Reconstructing full file paths using only the MFT is not trivial. Whether it is a reliable method at all is debatable. It is only possible because of an NTFS quirk - There's a backup of every file's `$FILE_NAME` attribute inside its MFT record. Nonetheless, this method is simple, fast, and is often good enough. In this post, I'll help you avoid its common implementation pitfalls.

## Before we Start

Given a `$MFT` file, we first have to determine the size of an individual file record. It's usually 1024 bytes, but it doesn't have to be. This value is stored in the file system boot sector - which we don't have. Instead, we can find any two consecutive file records, and calculate the difference between their offsets. This should give us the size of a file record.

Another thing to consider, is that file records, like other important file system structures, use [fixup values](https://flatcap.github.io/linux-ntfs/ntfs/concepts/fixup.html) to detect disk errors. If you ignore fixup values, your parser will output incorrect data, and the worst part is - you are probably not going to notice. Take my word for it, and start writing the code to deal with fixup values early on.

## A Naive  Approach

This is the structure of a `$FILE_NAME` attribute: 

| Offset | Size | Description                               |
| ------ | ---- | :---------------------------------------- |
| 0x00   | 6    | Parent index                              |
| 0x06   | 2    | Parent sequence                           |
| 0x08   | 8    | Creation time                             |
| 0x10   | 8    | Modification time                         |
| 0x18   | 8    | MFT record change time                    |
| 0x20   | 8    | Access time                               |
| 0x28   | 8    | Allocated size                            |
| 0x30   | 8    | Real size                                 |
| 0x38   | 4    | Flags                                     |
| 0x3C   | 4    | Used by EAs and Reparse                   |
| 0x40   | 1    | Filename length in bytes / 2 (L)          |
| 0x41   | 1    | Filename namespace                        |
| 0x42   | 2L   | Filename in Unicode (not null terminated) |

As mentioned earlier, each file has a `$FILE_NAME` attribute inside its MFT record. For now, we're only interested in these 2 fields:

* `Filename`  
  The name of the file described by the record.
  
  
  
* `Parent index`  
  An index to the MFT record of the file's parent folder.



Using these fields, we can build the full path of a file:

```markdown
                +--------------+
    +---------->| .            |    4. .\Documents\PDF\document.pdf
    |           +--------------+        
    |  +------->| Documents    |    3. Documents\PDF\document.pdf
    +--|--------+--------------+
       |        |              |
       |        +--------------+
       |  +---->| PDF          |    2. PDF\document.pdf
       +--|-----+--------------+
          |     |              |
          |     +--------------+
          |     |              |
          |     +--------------+
          |     | document.pdf |    1. document.pdf
          +-----+--------------+

In NTFS, "." is the file name of the root directory.
```



By looping over the MFT records, we can do this for every file on the system. Here's a Python-style pseudo code of the algorithm:

```python3
for current_record in mft.get_records():
    print(build_path(mft, current_record))

def build_path(mft, current_record):
    path = ""
    while not is_root_directory(current_record):
        filename_attribute = current_record.get_attribute("$FILE_NAME")
        path = "\\" + filename_attribute.get_field("Filename") + path
        current_record = mft.get_record_at(filename_attribute.get_field("Parent index"))
        
	return "." + path
```



Of course, we could have used caching to improve the efficiency - but for the sake of simplicity, all of the code in this post will be pretty inefficient. In the next sections, we'll see how deleted files, hard links and extension records break everything, and we'll change our code to deal with them.

## Pitfall 1: Orphan Files & the Sequence Number

The first edge case we'll discuss involves deleted files. When a file is deleted, its MFT record is marked as free. Then, it can be reused if a new file is created - why add a new record to the MFT when there's a free, existing one? If there are multiple free records, and a new file is created - it may occupy either one of them.

We can tell whether a record is in use by looking at the `Flags` field in the record header, and more specifically - at its least significant bit, which is the flag representing the state of the record. When a record is allocated to a file, it is set. When that file is deleted, it is turned off.

### Orphan Files

As long as the record of a deleted file is not reused - it remains intact, which means we can resolve the file's path just like any other file. Problems may arise if the file's parent folder is deleted as well. Let's see what happens when the parent folder's record is reused, while the child's record remains free:

1. Folder1 is created. It occupies the MFT record at index 41

   | Record state    | Record index | Reconstructed file path |
   | --------------- | ------------ | ----------------------- |
   | `In use`        | 41           | .\Folder1               |


2. File1.txt is created inside Folder1. It occupies the MFT record at index 42

   | Record state    | Record index | Reconstructed file path |
   | --------------- | ------------ | ----------------------- |
   | `In use`        | 41           | .\Folder1               |
   | `In use`        | 42           | .\Folder1\File1.txt     |


3. Folder1 is deleted. File1.txt is in Folder1, so it is also deleted

   | Record state   | Record index | Reconstructed file path  |
   | -------------- | ------------ | ------------------------ |
   | `Free`         | 41           | .\Folder1                |
   | `Free`         | 42           | .\Folder1\File1.txt      |


4. Folder2 is created. It occupies the MFT record at index 41

   | Record state | Record index | Reconstructed file path |
   | ------------ | ------------ | ----------------------- |
   | `In use`     | 41           | .\Folder2.txt           |
   | `Free`       | 42           | .\Folder2\File1.txt     |




File1.txt was never in Folder2!

Deleted files their parent folder's record was reused are called *Orphan Files*. They were once created in some folder, but it is now gone. their original path cannot be resolved. A folder can be orphan too, which will make it the root of an orphan sub-tree. The hierarchy of the files inside an orphan sub-tree can be reconstructed, but it's detached from the main directory tree. the original path of the root of the sub-tree, which is an orphan folder, cannot be resolved.

### The Sequence Number

On its own, an MFT index is not enough to reference a file. It references a record, which can be reused to describe many files over time. We don't want to reference records, we want to reference the files they describe. To reference a file, not only do we need to know its MFT index, but also whether the record in that index still belongs to it, or has been reused since.

This is where the sequence number comes in. It is stored in the record header, and is used together with the MFT index to reference files. That's why together, they are called a *File Reference*. Every time a record is **freed**, its sequence number is increased by 1. Then, any file references with the old sequence number are considered invalid.

Here's a pseudo-code of a reference validation function:

```python3
def is_valid_file_reference(mft, mft_index, sequence_number):
	file_record = mft.get_record_at(mft_index)
	if file_record.is_used():
	    return file_record.get_sequence_number() == sequence_number
	else:
        # the sequence number was increased when the record was freed, 
        # so we should decrease it to get the actual sequence number of the file
        return file_record.get_sequence_number() - 1 == sequence_number
```



Look again at the structure of the `$FILE_NAME` attribute. Not only do we have the MFT index of the parent folder, but also its sequence number. In other words - we have a file reference to the parent folder. This is exactly what we need to identify orphan files - all we need to do is validate this reference. If it's not valid, it means the parent folder was deleted, and its record was reused.

Now that we can identify orphan files, we have to decide how our code should deal with them. We could output the filenames and paths separately, and just not output a path for them. Or maybe we can place them all in some special folder, but which one? There's no folder it would be right to place them in. The solution we'll go with is quite simple: we'll just create a new folder for them.

Of course, it won't be an actual file system folder - we cannot create these. Nevertheless, we are the ones building the directory tree, so we can just "make up" a folder. It would be a "virtual folder", if you will. We'll call it "$OrphanFiles" and place it in the root directory. Now we have a foster parent for all the orphan files!

Here's our upgraded code:

```python3
for current_record in mft.get_records():
    print(build_path(mft, current_record))

    
def build_path(mft, current_record):
    path = ""
    while not is_root_directory(current_record):
        filename_attribute = current_record.get_attribute("$FILE_NAME")
        path = "\\" + filename_attribute.get_field("Filename") + path
        current_record = mft.get_record_at(filename_attribute.get_field("Parent index"))
		
        sequence_number = current_record.get_sequence_number()
        if not current_record.is_used():
            sequence number -= 1

        if sequence_number != filename_attribute.get_field("Parent sequence"):
            path = "\\$OrphanFiles" + path 
            break
        
    return "." + path
```

## Pitfall 2: Hard Links and Short Names

### Hard Links

Hard links are an often overlooked feature of NTFS. They are similar in functionality to the hard links in other file systems - a file can be located in multiple folders at the same time, and can have a different name in each one. Each directory index a file is located in stores a `$FILE_NAME` attribute for it. Fortunately for us, we have copies of all of these `$FILE_NAME` attributes in the file's MFT record.

This means a file can have multiple paths. To be exact, a file can have multiple hard links, and each one of *them* has a path. Each hard link has its own `$FILE_NAME`  attribute, containing its own file name and parent folder. To build all of the paths for a file, we'll loop over every `$FILE_NAME` attribute in its MFT record, and use them to build the path of each hard link separately.

Folders are easier to handle, because NTFS does not allow multiple hard links for them - it would allow you to create loops in the directory tree, and nobody wants that. This means a folder has only a single path. A hard link has a single parent folder - so it also has a single path. This means we'll build the path of each hard link very similarly to how we did it previously.

### Short Names

Short file names (SFNs) are a special type of hard links. If enabled, they're created automatically for files their names are too long to be handled by some old applications. They provide these old applications with an auto-generated, shorter file name they can handle. A short name is created in the same directory with the original hard link, and we can build the path for it the same way we do for a regular hard link.

We can tell whether a `$FILE_NAME` attribute contains a short name by checking its *Filename namespace* flag. A `DOS` namespace means it's a short name, and a `WIN32` or a `POSIX` namespace means it's a regular, long name. `WIN32 & DOS` is a special value indicating there is no short name for the file, because its name is already a valid short name.

Folders cannot have regular hard links, but they can have short names. This is a problem - if folders can have 2 different names, it means a hard link can have a huge number of different paths. Suppose a hard link is `n` folders deep in the directory tree, and each folder has 2 names at most - long and short, excluding the root directory, a hard link has at most `2ⁿ⁻¹` different paths!

```markdown
. \ ??? \ ??? \ ??? \ .... \ File.txt
     x2    x2    x2    ....
```

To keep things reasonable, we'll only use the long name of each folder to build the path of a hard link. By doing so, we're in fact choosing a single path for it, out of the many it may have. We can get away with it, because we build the paths for the folders in the exact same way. Each folder's short name is also a hard link, and we'll build a path for it when we get to that folder's MFT record. We're not loosing any short names.

Long names with the same prefix will have similar short names generated for them. It's important to output the MFT index of each path, so we can tell which short name belongs to which long name:

```markdown
MFT Index    Path
59:			.\Program Files
1293:		.\ProgramData
59:			.\PROGRA~1
1293:		.\PROGRA~2

PROGRA~1 --> Program Files
PROGRA~2 --> ProgramData
```



This is the upgraded code. It now handles hard links, and outputs both long and short names:

```python3
for current_record in mft.get_records():
    build_path(mft, current_record)


def build_path(mft, current_record):
	mft_index = current_record.get_mft_index()
    for filename_attribute in current_record.get_filename_attributes():
        print(mft_index + ": " + build_path_helper(mft, filename_attribute))


def build_path_helper(mft, filename_attribute):
    path = ""
    while True
        path = "\\" + filename_attribute.get_field("Filename") + path
        parent_record = mft.get_record_at(filename_attribute.get_field("Parent index"))
        if is_root_directory(parent_record):
            break
        
        parent_sequence = parent_record.get_sequence_number()
        if not parent_record.is_used():
            parent_sequence -= 1
        
        if parent_sequence != filename_attribute.get_field("Parent sequence"):
            path = "\\$OrphanFiles" + path 
            break
            
        filename_attribute = next(get_lfn_filename_attributes(parent_record))
        
    return "." + path


# Get only filename attributes with a Long File Name (LFN)
def get_lfn_filename_attributes(file_record):
    for filename_attribute in file_record.get_filename_attributes():
        if filename_attribute.get_field("Name space") != "DOS":
            yield filename_attribute
```



If you don't care about the short names, you don't have to build paths for them. You can simply ignore them altogether. That's perfectly fine, but I believe it's better to output both long and short names - and group them into a single event, later in the analysis phase. To ignore short names, all we need is a small modification to the `build_path` function:

```python3
def build_path(mft, current_record):
    for filename_attribute in get_lfn_filename_attributes(current_record):
        print build_path_helper(mft, filename_attribute)
```

## Pitfall 3: Extension Records, Missing Attributes and Orphaned Attributes

### Extension Records

If a file becomes extremely large and fragmented, or has many hard links, a single file record may not be big enough to contain all of its attributes. In such case, the file will occupy additional file records. The file's main record is called the *Base Record*, and the additional ones are called *Extension Records*. When some of a file's attributes are stored in extension records, its base record contains a special attribute, called `$ATTRIBUTE_LIST`, which stores the information needed to find them.

Some MFT parsers use the `$ATTRIBUTE_LIST` attribute in the base record to find its extension records. This is a major mistake, because `$ATTRIBUTE_LIST` attributes can be non-resident (stored outside the MFT). Instead, we'll take the opposite approach - find the extension records, and trace them back to their base record.

Conveniently, the file-record header contains a `Base index` and `Base sequence` fields, to store a file reference to the base record. In base records, these fields are zeroed out. This means we can also use them to differentiate base records from extension records, without having to check for a `$ATTRIBUTE_LIST` attribute.

By looping over the MFT, we can create a mapping of base records to extension records:

```python3
# a dictionay to map base records to their extension records
# mappings are of the form: (n, m) -> [k1, k2, ...], where:
# (n, m) is a file reference of a base record
# ki is the MFT index of an extension record
extension_records = {}


for current_record in mft.get_records():
    if not is_base_record(current_record):
        base_index = current_record.get_base_record_index()
        base_sequence = current_record.get_base_record_sequence()

        (extension_records
            .setdefault((base_index, base_sequence), [])
            .append(current_record.get_mft_index()))


def is_base_record(current_record):
    return (current_record.get_base_record_index() == 0 and
            current_record.get_base_record_sequeunce() == 0)
```

After we create this dictionary, we have to loop over the MFT one more time. This time, looking for base records. For each base record, we will build a path for all the `$FILE_NAME` attributes both in it, and in its extension records - if it has any.

### Missing Attributes

As always, deleted files create problems. When a file is deleted, its base record and all of its extension records are marked as free. The problem is - all of those free records are not going to be reused all at the same time. If some, but not all of the file's records are reused - we loose some, but not all of the file's attributes. This has major consequences on MFT parsing in general, and on path reconstruction in particular.

We may find a file without any `$FILE_NAME` attributes. For such a file, path reconstruction is impossible. The best you can do is give it a unique identifier. You may be tempted to place it in the $OrphanFiles folder, but that wouldn't be right -  it's not necessarily orphan. Sure, it's deleted, but its parent folder might not be.

If you find a **folder** without any `$FILE_NAME` attributes, you may also find deleted files that were once inside it. Those files can be safely placed in the $OrphanFiles folder, because the folder is deleted, and we won't output a path for it.

### Orphaned Attributes

To resolve the parent folder for a file, we validate the parent reference in its `$FILE_NAME` attribute. If it's not valid, it means the file is orphan, right? Well, yes - but we can do better than that. Before declaring the file is orphan, we should check whether its parent has any extension records left. If there's a `$FILE_NAME` attribute in one of them, we can use it to resolve the file's path.

This is the final version of our code. It handles everything we talked about so far: orphan files, hard links and extension records:

```python3
# a dictionay to map base records to their extension records
# mappings are of the form: (n, m) -> [k1, k2, ...], where:
# (n, m) is a file reference of a base record
# ki is the MFT index of an extension record
extension_records = {}


# first pass over the MFT
for current_record in mft.get_records():
    if not is_base_record(current_record):
        base_index = current_record.get_base_record_index()
        base_sequence = current_record.get_base_record_sequence()

        (extension_records
            .setdefault((base_index, base_sequence), [])
            .append(current_record.get_mft_index()))


# second pass over the MFT
for current_record in mft.get_records():
    if is_base_record(current_record):
        build_path(mft, current_record)


def is_base_record(current_record):
    return (current_record.get_base_record_index() == 0 and
            current_record.get_base_record_sequeunce() == 0)


def build_path(mft, current_record):    
    base_filename_attributes = current_record.get_filename_attributes()
    extension_filename_attributes = get_extension_filename_attributes(
        current_record.get_mft_index(),
        current_record.get_sequence_number()
    )
    
    mft_index = current_record.get_mft_index()
    for filename_attribute in base_filename_attributes + extension_filename_attributes:
    	print(mft_index + ": " + build_path_helper(mft, filename_attribute))


def build_path_helper(mft, filename_attribute):
    path = ""
    while True
        path = "\\" + filename_attribute.get_field("Filename") + path
        parent_record = mft.get_record_at(filename_attribute.get_field("Parent index"))
        if is_root_directory(parent_record):
            break
        
        parent_sequence = parent_record.get_sequence_number()
        if not parent_record.is_used():
            parent_sequence -= 1
        
        parent_filename_attributes = []
        # if parent record still belongs to my parent, get its filename attributes
        if parent_sequence == filename_attribute.get_field("Parent sequence"):
            parent_filename_attributes = parent_record.get_filename_attributes()
        
        # get the filename attributes from any extension records of my parent
        parent_filename_attributes += get_extension_filename_attributes(
        	filename_attribute.get_field("Parent index"),
            filename_attribute.get_field("Parent sequence")
        )

        if not parent_filename_attributes:
	        # could not find a $FILE_NAME attribute of my parent
            path = "\\$OrphanFiles" + path 
            break

        # prioritize LFNs when choosing a name for the parent
        filename_attribute = max(parent_filename_attributes, key=get_priority)        
        
    return "." + path


def get_extension_filename_attributes(base_index, base_sequence):
    extension_records =  map(
    	mft.get_record_at, 
    	extension_records[(base_index, base_sequence)]
    )
    
    filename_attributes = []
    for record in extension_records:
        filename_attributes += record.get_filename_attributes()
    
    return filename_attributes


def get_priority(filename_attribute):
    if filename_attribute.get_field("Name space") == "DOS":
        return 0
    else:
        return 1
```

In the previous version, we built paths using the long file name (LFN) of each folder, because the long names are more meaningful than the auto-generated short names. Now we know that the `$FILE_NAME` attribute containing the short name may be the only `$FILE_NAME` attribute left after some of a folder's records were reused.

In this new version, we prioritize the LFN `$FILE_NAME` attributes, but we will use the SFN `$FILE_NAME` attribute if it's the only one left.

## Closing Thoughts

This post was my attempt to create the resource I wish I had when I wrote my MFT parser. MFT parsing is such a fundamental technique in forensics; yet, the resources to learn it are lacking. People have been writing MFT parsers for years, each one on their own, rediscovering the same edge cases and learning the same lessons again and again.

It really shouldn't be like that. I think we DFIR programmers have a lot to learn in sharing our knowledge with one another, so we can push the field forward, together.

