# Ubuntu server hardening, performance and Docker

[Follow the step by step walkthrough with screenshots](WALKTHROUGH.md)

I ran two Ubuntu virtual machines in VirtualBox. I used the desktop VM as my client and logged into the server over SSH. That gave me two views of the same machine. On the server I could see which processes were listening. From the client I could check which ports another machine could actually reach.

## Reducing the services exposed by the server

My first Nmap scan found SSH on port 22 and Apache on port 80. I had used Apache for load testing, but the server only needed SSH for remote administration. I took a snapshot, disabled Apache and ModemManager, and configured the firewall to allow SSH. I then repeated the TCP scan from the client. Only port 22 appeared open.

I used these checks to compare the external scan with what the server itself reported.

```bash
# On the client
nmap -p- -sV 10.0.2.3

# On the server
sudo ss -tulnp
findmnt -no TARGET,OPTIONS /dev/shm
```

I checked the shared memory mount after applying the restrictions and found `nosuid`, `nodev` and `noexec`. I also tightened password quality and set password ageing for my lab account. Lynis went from **57 to 61 out of 100**. Suggestions fell from 50 to 45, with one warning still open. I made several changes between audits, so the score only tells me their combined effect.

One finding mattered more to me than the score. The audit daemon was running, but `sudo auditctl -l` showed no rules. My next test would be to add persistent rules for changes to the account database and SSH configuration, make a controlled change and check that I could find the event afterwards.

## Measuring storage and web traffic

I used `stat` to compare two hard links and a symbolic link. Both hard links showed inode 917515. The symbolic link had its own inode, 917516. On the ext4 root volume I checked free space and inode usage separately because creating lots of small files can use up the available inodes.

I wrote a 1 GiB test file with `dd`, cleared the guest cache and read it twice. The direct write reached 101 MB/s. The first read took 3.31 seconds at 324 MB/s and the immediate repeat took 0.96 seconds at about 1,100 MB/s. The repeat showed how much the cache could affect a benchmark. Clearing the guest cache still left the host cache as a possible influence.

I also sent 1,000 requests to Apache from the client. Raising concurrency from 10 to 100 took throughput from about 1,729 to 5,462 requests per second. Mean time per request also rose from 5.79 to 18.31 ms. Both runs finished in under a second, so I would use longer runs to judge sustained performance.

## Building and testing containers

I ran two Nginx containers. The second one initially failed because I had already mapped host port 8080. Giving it port 8081 let both containers serve their welcome pages on the same VM.

I then built an Ubuntu image called `sysinfo` with a shell script that printed the hostname, kernel, process count and memory. My first image built successfully but failed when I ran it. Fixing the Dockerfile command to launch the script with Bash got it working. When I changed the hostname label for version 1.1, Docker reused the package installation layer and rebuilt the layers that depended on the script.

```bash
sudo docker run --rm sysinfo:1.0
sudo docker history sysinfo:1.0
```

Finally, I tested CPU limits of 0.25, 0.50 and 1.00 on a VM with one virtual CPU. Docker reported 397.78%, 431.15% and 393.34% in the last run. Those figures were implausible on this setup, and the VM had also reported timing stalls. I kept the readings as a troubleshooting result. To measure the limits reliably, I would repeat the test on a stable host and compare CPU time with elapsed time.

