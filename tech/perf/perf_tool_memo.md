perf的基本使用

```sh
$ perf stat -e r<UnitMask(16進数)><EventSelect(16進数)>:[u/k] ./a.out 100000
```

基本的perf测试程序脚本

```python
#!/usr/bin/env python
# This script is written referring to
#   Basic Performance Measurements for AMD Athlon 64,
#   AMD Opteron and AMD Phenom Processors
#   http://developer.amd.com/Assets/intro_to_ca_v3_final.pdf
import sys
import re
import subprocess

# Modify this list as you like.
# The format of each element is:
#   select: `NNN' for "perf stat -e rNNN"
#   name:   Event name used in output
events = [
  {"select": "c0", "name": "Retired Instructions"},
  {"select": "40", "name": "Data Cache Accesses"},
  {"select": "1e42", "name": "Data Cache Refills from L2"},
  {"select": "1e43", "name": "Data Cache Refills from System (Northbridge)"}
]

def event_list():
  ret = ""
  for event in events:
    ret += "-e r" + event["select"] + "   "
  return ret

def store_count_to_events(fd_perf_output):
  # Sample perf-stat output line
  #        1565650  raw 0xc0                 #      0.000 M/sec
  pat = re.compile("^ *([0-9]+).*")
  lines = fd_perf_output.readlines()
  for event in events:
    search_str = "raw 0x" + event["select"]
    event_line = [line for line in lines if line.find(search_str) != -1]
    mat = pat.match("".join(event_line))
    event["count"] = int(mat.group(1))
    
def print_events():
  for event in events:
    print "%10s   %s" % (str(event["count"]), event["name"])
    
def main():
  command = "perf stat " + event_list() + " ".join(sys.argv[1:])
  p = subprocess.Popen(command, shell=True,
                       stderr=subprocess.PIPE)
  fd_perf_output = p.stderr
  store_count_to_events(fd_perf_output)
  print_events()
  
if __name__ == "__main__":
  main()
```