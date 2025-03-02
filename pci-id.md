<!-- GNU Affero General Public License v3.0 or later (see COPYING or https://www.gnu.org/licenses/agpl.txt) -->
```
$ oc debug node/vulcan.internal
Starting pod/vulcaninternal-debug-7qbqc ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.1.88
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host

sh-5.1# lspci -nnv | grep -i radeon
03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 [Radeon RX 7900 XT/7900 XTX/7900M] [1002:744c] (rev c8) (prog-if 00 [VGA controller])
6d:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Rembrandt Radeon High Definition Audio Controller [1002:1640]

sh-5.1# lspci -nnv | grep -e "HDMI/DP Audio"
03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 HDMI/DP Audio [1002:ab30]
        Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 HDMI/DP Audio [1002:ab30]
```