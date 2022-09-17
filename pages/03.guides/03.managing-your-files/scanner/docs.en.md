---
title: Scanner
taxonomy:
    category:
        - docs
visible: false
---

>>> This guide is a WIP writeup of the new Scanner dedicated page

## Page overview
- [Introduction](#introduction)
  - [What happens during a Scan](#what-happens-during-a-scan)
  - [Refresh Covers](#refresh-covers)
  - [Analyze Files](#analyze-files)
- [File Layout](#file-layout)
- [The Scan Loop](#the-scan-loop)
- [Notes](#notes)

<hr style="border:5px solid #4ac694">

## Introduction
Scanning a library makes Kavita check its folders and sub-folders for new or removed items (books, archive files, etc). If new media is found, it then pulls it into the library. <br/>You can think of scanning as “check for new or changed content”.

! **Important**:<br/>- First scans are often slow, especially on networked storage. Be patient<br/>- The Kavita Homepage and Library info, will be updated throughout the scan

<hr style="border:2px solid #4ac694">

### What happens during a Scan?
Kavita will generate a library representation of your files on disk. A Kavita library does _not_ represent exactly your folder structure. Kavita uses filenames, internal metadata, and some limited folder names to parse out the series, volume, chapter, etc from the file and group them.

The scan parses the file names, reads the comic info (if applicable), updates the database with that information, and updates the UI. 
If the file hasn't been modified since the last time Kavita scanned, it will not do extra procesings on the file. 
If your archives contain metadata, it will override any parsed information from the file.

To understand in depth how Kavita's scan works, go to the scan loop section of this page [here](#The Scan Loop).

<hr style="border:2px solid #4ac694">

### Refresh Covers
During the refresh covers task, the same kind of logic applies. This is a heavy task because of the amount of I/O Kavita has to perform and because of the amount of memory it needs to copy images out of the archive and onto the disk.
In this task, Kavita doesn't open up any archives if they haven't been modified unless you start a cover refresh from the UI. Even if the archive was modified, if you've locked the cover image by using the UI to upload your own custom cover, the archive will not be opened.


<hr style="border:2px solid #4ac694">

### Analyze Files
During the analyze files task, Kavita will open epub files and count the number of words per entity. This is I/O and memory intensive. Like other tasks, Kavita employs checks against Last Modified to avoid re-calculation whenever possible. When invoking this task manually from the UI, it will force a recalculation, so be very careful if you use remote storage or a slow server.

<hr style="border:5px solid #4ac694">

# File Layout
Kavita expects all series to be nested in a folder. The ideal layout is:
```
Library Root
  ┠── Series Name A
  │   ┠── Series Name A - v01.cbz
  │   ⋮
  │   ┠── Series Name A - v06.cbz
  │   ┖── Specials
  │     ┖── Artbook 1.cbz
  │
  ┖── Series Name B
      ┠── Series Name B - v01.cbz
      ⋮
      ┠── Series Name B - v06.cbz
      ┖── Specials
        ┖── Artbook 1.cbz
```

This means you can also have:
```
Library Root
  ┠── Publisher A
  │   ┠── Series Name A
  │   │   ┠── Series Name A - v01.cbz
  │   │   ⋮
  │   │   ┖── Series Name A - v06.cbz
  │   │
  │   ┖── Series Name B
  │             ┖── Oneshot.cbz
  │
  ┖── Publisher B
      ┠── Series Name C
      │   ┠── Series Name C - v01.cbz
      │   ⋮
      │   ┖── Series Name C - v06.cbz
      │
      ┖── Series Name D
                ┖── Oneshot.cbz
```
[//]: # (;wiki dev note: Should we add some common not supported structures?&#41;)

!! But no files can exist at root level

!! Series cannot be between 2 adjacent folders (aka  `Series Name B`  cannot have something from  `Series Name A` ). **From the library root**
```
Library Root
  ┠── Series A
  │   ┖── Series A File 1.cbz
  ┖── Series B
      ┖── Series A File 2.cbz
        
```
If these rules are followed, you shouldn't have any problems.

<hr style="border:5px solid #4ac694">

# The Scan Loop
In-depth overview on how the scan loop works

<hr style="border:2px solid #4ac694">

## Step 1: Validate if Kavita needs to scan
- On all scans, Kavita will validate that all library folders are not empty and can be accessed. If they cannot, the scan is aborted. For a series, Kavita also checks the folder path on disk for the series exists, if it does not, the scan gets aborted. 

<hr style="border:2px solid #4ac694">

## Step 2: Scan the directories
- The scanner is pretty smart, it avoids as much work as possible by default. 
- For each folder (in a library or for a given series), Kavita first checks if the folder has changed since our last scan. This checks at a minute level, so seconds will be ignored. This validates on the Last Write Time of the folder. (Note: Last Write time requires scanning all nested folders to accurately report the last time that folder has changed)
- In each folder, Kavita checks for a .kavitaignore and parse if it exists. This allows Glob patterns to apply to the scanner and ignore certain files or folders. This applies recursively. See more [here]().
- Files that are ignored in `.kavitaignore` will be excluded from the scan. [See kavitaignore usage](/guides/misc/ignoring-files-and-folders)

<hr style="border:2px solid #4ac694">

## Step 3: Processing files
- The library dictates the rules for the parser and there are different parsers for different file types. 
- The parser parses out any information from the files, these files are a list of files collected from the folder scan in the previous step and are assumed to be one series
- After parsing, Kavita checks any series need to be merged together. Merging might happen if there are ComicInfo's with `LocalizedSeries` tag which allows 2 different names to be merged together automatically. Note: If there are multiple series in one folder with localizedSeries tag, they will group incorrectly. Kavita will log this, but not stop the scan. This is not a valid configuration.
- For each found series in the folder (should be one), Kavita invokes a process series task

<hr style="border:2px solid #4ac694">

## Step 4: Process Series
- This is responsible for taking the processed data and updating the Database. This runs parallel with other process series tasks. This may make the logs difficult to understand.
- The first thing that needs to happen is find the series in the Database. This is done by checking against Series name and localized name. If neither exist, a new series will be created.
- In here, Kavita does all the underlying db work and update fields, etc.
- Lastly Kavita queues tasks to perform Cover Generation and File Analysis. These again will run in parallel and are scheduled on the same queue as other scan tasks. 

>>> If there are no modifications to folders, Kavita will not scan or process them.

<hr style="border:5px solid #4ac694">

# Notes
- Having multiple series in one folder is not supported but does work. There are some caveats to this. If there exists a series in that folder that utilizes LocalizedSeries ComicInfo tag, then the series may group in an undexpected way. This will be informed to the user via the Log file, ie ``. In addition, Folder Watching wont pick up on series changes correctly due to utilizing the same folder. 
- For series scan, if the series folder is no longer on the disk, the scan will be aborted. A library scan should be run which will delete the series. 
- Not every action you can perform on a folder will change its modification time, including renaming and moving it. Keep this in mind if library scans are not working as expected.