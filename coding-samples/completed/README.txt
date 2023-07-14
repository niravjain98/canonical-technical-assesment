Copy your new script here

#I have chosen that second script that collects some information 
#about a disk to ensure that it's recognized by the operating system and 
#that it's represented accurately.

#Python script equivalent of the second bash script.

#To run this Python script, type python3 script_name.py [disk_name]

import os
import sys
import time
import subprocess

class Disk:
    def __init__(self, name):
        self.name = name
        self.start_stats = {
            'proc': None,
            'sys': None,
        }
        self.end_stats = {
            'proc': None,
            'sys': None,
        }

    def gather_stats(self, start=True):
        stage = 'start' if start else 'end'
        with open(f'/proc/diskstats') as file:
            for line in file:
                if self.name in line.split():
                    self.start_stats['proc'] = line.strip() if start else None
                    self.end_stats['proc'] = None if start else line.strip()
        with open(f'/sys/block/{self.name}/stat') as file:
            self.start_stats['sys'] = file.read().strip() if start else None
            self.end_stats['sys'] = None if start else file.read().strip()

    def generate_activity(self):
        subprocess.call(['hdparm', '-t', f'/dev/{self.name}'], stdout=subprocess.DEVNULL, stderr=subprocess.STDOUT)
        time.sleep(5)


def check_disk(disk):
    if 'pmem' in disk.name:
        print(f"Disk {disk.name} appears to be an NVDIMM, skipping")
        return False
    if not os.path.exists(f'/sys/block/{disk.name}'):
        print(f"Disk {disk.name} not found in /sys/block")
        return False
    if not os.path.exists(f'/sys/block/{disk.name}/stat') or os.path.getsize(f'/sys/block/{disk.name}/stat') == 0:
        print(f"stat is either empty or non-existent in /sys/block/{disk.name}")
        return False
    return True


def main():
    status = 0
    disk_name = sys.argv[1] if len(sys.argv) > 1 else 'sda'
    disk = Disk(disk_name)
    if check_disk(disk):
        disk.gather_stats(start=True)
        disk.generate_activity()
        disk.gather_stats(start=False)
        if disk.start_stats['proc'] != disk.end_stats['proc']:
            print("Stats in /proc/diskstats did not change")
            status = 1
        if disk.start_stats['sys'] != disk.end_stats['sys']:
            print("Stats in /sys/block/{disk.name}/stat did not change")
            status = 1
        if status == 0:
            print(f"PASS: Finished testing stats for {disk.name}")
    else:
        status = 1
    sys.exit(status)


if __name__ == '__main__':
    main()

