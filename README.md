slow-disk-check
================

Background
-----------

This tool was designed to help analyze and diagnose block devices that have random slow points.

In particular, a spinning disk, which is typically capable of 30-100MB/sec sequential read rates, can occasionally drop down to less than 1MB/sec in areas that are weak or defective.  Sometimes, these are remapped sectors on a disk hindered by seek times. Sometimes these are indicative of a damaged disk, and may identify the nature of the damage - potentially platter or head defects on spinning media, or weak areas of flash on solid state disks.  In any case, these are often areas that one would want to avoid when considering performance-centric applications, or even for day-to-day operations because it can bring the system to a crawl if this happens to be amid your NTFS MFT or other critical system areas of the filesystem.

This tool helps map out the disk's performance restricted areas, and was developed with several thoughts in mind:
- I was not able to find any simple tools for Linux that map out a disk's read performance across the range of its entire surface.  This would be similar to ATTO or Passmark's disk test that shows read rates as a graph across the disk's addressable range.
- I wanted to map out slow sectors on a disk in a knock-around system, in hopes of not having to replace the disk and simply repartitioning the disk around the bad areas.
- One can use data from a simple command line tool to generate graphs in any spreadsheet application, so I didn't focus on creating a GUI or using a graphing toolkit.


Usage
------

The tool is rather simple to use

`slow-disk-check [options] block_device`

options can be any of:
- `-f step_size_in_MB` - the "fast scan" step size used to move across the disk faster.  If you want to literally scan every sector, use a value of 1, but typically, 50-100 is more than enough and significantly reduces the scan time.  For benchmarking purposes, a value of 1000 or more can be ideal.
- `-t slow_time_in_ms` - the "slow time" in ms.  After 2 attempts to read 1MB that take longer than this time, the tool switches to "slow detection mode" at which point it attempts to find the beginning and ending address of the "slow spot" range.  This is currently tuned for a spinning disk, and will likely require some tuning for an SSD.
- `block_device` - Should be a typical block device, like `/dev/sda`.  Obviously you will need read access to the device as the user running this tool, which often means running this tool as root.  This obviously comes with risks, but you can read the source code if you want to be safe.


Output
-------

Output as it runs is generally of the form:

```
(  2%)    2400 MB: 306 ms = 3 MB/s, avg=42 MB/s: 0.374117 s, 2.8 MB/s
```
```
\____/    \_____/  \____/   \____/  \_________/  \________/  \______/
  |          |       |        |          |           |          |
  1          2       3        4          5           6          7
```
1. Percent complete in scanning the entire device.
   - `(SLOW)` indicates that the tool is performing slow region analysis to find the beginning and end of the slow region.
2. LBA of the current scan location.  This is the basis for the %.
3. Time it took to read 1MB from the current scan location
4. Rough read rate for that 1MB.  This is an integer calculation, so some rounding may occur.
5. Average read rate for non-slow addresses.  This statistic automatically excludes ratings from any regions detected as slow, so this could be seen as an ideal read rate.
6. Time to read 1MB according to the low-level dd call
7. Read rate according to the low-level dd call


It can be worthwhile to log the scan for later data analysis.  One can do this with a command like:

`slow-disk-check /dev/sdX | tee /path/to/log_file`

For those who are not already aware, this will display the output on screen and also log it to the specified file, as the system's tee tool is designed to do.


Post-Processing
----------------

there is also a parser included to extract useful information from recorded log files into tabular data for use in a spreadsheet or another tool.

Use it as:

`parse-scan-log log_file`

It will automatically generate `log_file.rates` and `log_file.slowranges` in the same directory as the original `log_file`.  These files will have tab-separated fields, which is ideal for copy/pasting into a spreadsheet application.

- `log_file.rates` will contain an entry for every read done on the device, in order. with the following fields:
  - LBA address
  - time in ms to read 1MB at that LBA address
  - time in seconds to read 1MB at that LBA address as reported by the low-level dd command
- `log_file.slowranges` will contain an entry for each `SLOW:` line in the output, identifying slow regions by LBA address.
  - LBA start address of the slow region
  - LBA end address of the slow region

The `log_file.rates` data is great for mapping the read speeds across the entire disk.  This could also be useful in benchmarking.

The `log_file.slowranges` data is best for finding address patterns in the slow areas to help identify the cause of the slowness.  For example, repeating slow areas at a nearly fixed interval, could indicate a failing head or platter.  A few large blocks could indicate a bit of physical damage the disk sustained.


Summary
--------

In the initial case that inspired me to create this tool, I was not able to work around the problem, but figured the tool may well be useful in the future to analyze other disks.  After seeing the patterns in the data this tool provided, I determined the disk in question must have a bad platter or head, and decided to simply replace the disk to restore typical performance on the affected system.

The testing done by this tool is read-only, so it should not damage any data directly, but using any software tools on damaged disks has the potential for data loss, as accessing the disk may cause it to fail faster or more severely.  The best solution for data recovery is always handing it to a professional.

As with any free software, there is no warranties of any kind with this tool.  If you use this on critical data and it causes you any kind of harm or inconvenience (directly or indirectly), I am not responsible.

