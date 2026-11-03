# Incident 01: Disk Space Not Freed After `rm` (Deleted-but-Open File)

**Platform:** Rocky Linux VM 
**Category:** Production troubleshooting, disk and storage
**Skills practised:** `df`, `fallocate`, `rm`, `lsof`, background jobs, `kill`

---

## 1. Summary

A 1 GB file was created and then deleted with `rm`, but `df -h /` still reported the space as used. The space was held by a running process that still had the file open. Once that process was terminated, the space was released and usage returned to baseline.

**Key lesson:** `rm` removes only the filename (the directory entry). The disk blocks are freed only when the last process holding the file open closes it.

---

## 2. Timeline

| # | Action | Command | Result |
|---|--------|---------|--------|
| 1 | Record baseline | `df -h /` | 1.8G used, 20G available, **9%** |
| 2 | Create a 1 GB file | `fallocate -l 1G /var/tmp/bigfile.log` | File created |
| 3 | Check usage | `df -h /` | 2.9G used, 19G available, **14%** |
| 4 | Hold the file open in the background | `tail -5 /var/tmp/bigfile.log &` | Job `[1]`, PID `15732` |
| 5 | Delete the file | `rm /var/tmp/bigfile.log` | No error |
| 6 | Check usage again | `df -h /` | Still 2.9G used, **14%** (space not freed) |
| 7 | Find the holder | `sudo lsof +L1` | `tail` (PID 15732) holds `/var/tmp/bigfile.log (deleted)` |
| 8 | Terminate the holder | `kill %1` | `[1]+ Terminated` |
| 9 | Verify | `df -h /` | 1.9G used, 20G available, **9%** (recovered) |

---

## 3. Evidence

**Baseline (before the incident):**

![Baseline df output](images01-baseline-df.png)

**Full incident, from file creation to recovery:**

![Incident sequence](images02-incident.png)

---

## 4. Diagnosis

After `rm`, `df` still showed 14% used. The next step was to look for deleted files that are still open:

```bash
sudo lsof +L1
```

The relevant line of output:

```
tail   15732   nxg   3r   REG   253,0   1073741824   0   50734619   /var/tmp/bigfile.log (deleted)
```

| Column | Value | Meaning |
|--------|-------|---------|
| COMMAND | `tail` | The program holding the file |
| PID | `15732` | Process ID, matches the job started in step 4 |
| USER | `nxg` | Owner of the process |
| FD | `3r` | File descriptor 3, opened for reading |
| DEVICE | `253,0` | The real disk (LVM volume), not RAM |
| SIZE/OFF | `1073741824` | 1 GiB, matches the file created in step 2 |
| NLINK | `0` | No directory entries remain, so the file has no name |
| NAME | `... (deleted)` | The path it had before `rm` |

### Other lines in the output (noise)

```
dbus-broker  /memfd:dbus-broker-log (deleted)
firewalld    /memfd:libffi (deleted)
```

These are normal. `memfd` files are anonymous, RAM-backed files created by programs on purpose. They have no disk usage and are not related to this incident. Hide them with:

```bash
sudo lsof +L1 | grep -v memfd
```

---

## 5. Root cause

The file was deleted while process `tail` (PID 15732) still had it open. Linux keeps the data on disk until every open file descriptor is closed, so `df` kept counting the 1 GB as used even though `ls` could no longer see the file.

---

## 6. Fix and verification

```bash
kill %1
df -h /
```

Terminating the process closed the file descriptor and released the blocks. Usage returned from 14% to 9% (1.9G used, about 0.1G above the baseline, which is normal system activity such as logs).

---

## 7. How this is handled in production

Killing a process is not always possible, for example a database or web server holding a deleted log file. In that case, empty the file through the process's file descriptor instead:

```bash
sudo truncate -s 0 /proc/<PID>/fd/<FD>
```

Use the PID and FD number from the `lsof` output (for `3r`, use `3`). The space is released and the process keeps running.

---

## 8. Commands and flags used

| Command | Flags and arguments explained |
|---------|-------------------------------|
| `df -h /` | `df` reports filesystem usage. `-h` prints human-readable sizes (G, M). `/` limits the report to the filesystem holding the root directory. |
| `fallocate -l 1G <file>` | `fallocate` reserves disk space instantly without writing data. `-l 1G` sets the length to 1 gigabyte. |
| `tail -5 <file> &` | `tail` prints the end of a file. `-5` selects the last 5 lines. `&` runs the command in the background and returns the prompt. |
| `rm <file>` | Removes the directory entry (the name). Disk blocks are freed only when no process holds the file open. |
| `sudo lsof +L1` | `sudo` runs as root so all users' processes are visible. `lsof` lists open files. `+L1` shows only files with a link count below 1 (deleted but still open). |
| `kill %1` | Sends SIGTERM to background job number 1. `%1` is a shell job reference, not a PID. |
| `truncate -s 0 <path>` | `truncate` changes file size. `-s 0` sets it to zero bytes. |

---

## 9. Notes and limitations

- `tail -5` was used instead of `tail -f` to hold the file open. On a normal log file, `tail -5` prints five lines and exits almost immediately, so no ghost file would exist. Here it stayed running long enough to be caught because the file is 1 GB of zeros with no line breaks, which forces `tail` to scan a very large amount of data. This makes the reproduction timing-dependent. Use `tail -f` (`-f` = follow, keeps the file open) for a reliable reproduction.
- The first `sudo` attempt showed "Sorry, try again", which was a mistyped password. 

---

## 10. Interview answer (30 seconds)

> "If `df` shows the disk as full but `du` can't find the files, I check for deleted-but-open files with `lsof +L1`. Deleting a file only removes its name. The space comes back when the last process using it closes it. I'd restart the process, or truncate the file through `/proc/<PID>/fd/<FD>` if the service can't be restarted."

---

## 11. Reproduce it yourself

```bash
df -h /                                   # 1. baseline
fallocate -l 1G /var/tmp/ghost.log        # 2. create the file
tail -f /var/tmp/ghost.log &              # 3. hold it open (-f follows the file)
rm /var/tmp/ghost.log                     # 4. delete it
df -h /                                   # 5. space is NOT freed
sudo lsof +L1 | grep -v memfd             # 6. find the holder
kill %1                                   # 7. release it
df -h /                                   # 8. space is freed
```
