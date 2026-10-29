---
title: "Peak vs. Persistence: A Guide to Performance and Burn-In Testing"
date: 2024-08-19
medium_url: https://medium.com/@rabbit_knight/peak-vs-persistence-a-guide-to-performance-and-burn-in-testing-284cb430cce6
---

> **Original Published Article on Medium:** [Peak vs. Persistence: A Guide to Performance and Burn-In Testing](https://medium.com/@rabbit_knight/peak-vs-persistence-a-guide-to-performance-and-burn-in-testing-284cb430cce6)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*FC1YYcK8to7noF-svqRRrA.jpeg)

### Peak vs. Persistence: A Guide to Performance and Burn-In Testing

### Performance Test

A Performance Test is designed to measure your system’s peak capabilities. It does not assess the system’s resilience or reliability under prolonged stress. For that purpose, you’ll need to conduct a Burn-In Test.

#### Steps to Run a Performance Test

1. Insert Flash Drive: Make sure you have a bootable USB drive with Ubuntu ( link to how to create a bootable USB) . UEFI will prioritize booting from the flash drive, speeding up the process.
2. Power On: Turn soft-power on and wait for the beep and the Grub prompt to appear.
3. Select Boot Option: Choose “Try or Install Ubuntu” to load the Ubuntu Gnome UI.
4. Network Configuration: If using Ethernet, plug in the cable. For Wi-Fi, navigate to “Settings → WiFi,” select your network, and enter the password.
5. Launch Firefox: Open Firefox to access the internet.
6. Download PerformanceTest: Search for the PassMark website on Google and download PerformanceTest.
7. Select Architecture: Choose the appropriate CPU architecture and OS version.
8. Extract Files: Unzip the archive and refer to the README for instructions.
9. Navigate to Folder: Go to the directory where you downloaded the PerformanceTest.
10. Run the Test: Use the command ./pt_linux_<arch> , replacing <arch> with your CPU architecture (likely x86 for most users).
11. Handle Errors: If the program fails to run due to a missing libncurses.so.5 , proceed to the next step.
12. Check Libraries: Run find / -name "libncurses.so*" to locate any installed versions of the library.
13. Create a Symlink: If you find a different version of the libncurses library (e.g., libncurses.so.6 ), create a symbolic link in the same directory where the original file resides. This allows the PerformanceTest to use the existing version. Use the following command:

```
sudo 
ln
 -s /usr/lib/…/libncurses.so.6 /usr/lib/…/libncurses.so.5
```

14. Run the Program: After creating the link, increase the terminal window size for better visibility, and re-run the PerformanceTest.

15. Start the Test: Press Shift-A to initiate all performance tests. You should hear the system’s cooling fans ramp up as the tests begin.

16. Compare Results: Once the test completes, visit cpubenchmark.net to compare your system’s performance against similar setups. This comparison will help you determine if your system is configured correctly and whether there are any potential bottlenecks in your build.

17. Verify CPU:

- Run sudo dmidecode | less to check your CPU details in Linux.
- Copy your CPU name and search for it in the CPU Mega List to ensure your system is performing as expected.

### Burn-In Test

A Burn-In Test evaluates the long-term reliability and resilience of your system under sustained high loads. It does not measure peak performance or identify bottlenecks; for that, use the PerformanceTest.

#### Steps to Run a Burn-In Test

1. Download and Unpack: Search for the BurnInTest for Linux, download it, and extract the files.
2. Navigate to the Folder: Open your terminal, navigate to the directory where you unpacked the tool, and prepare to run the test.
3. Run the Test Script: Execute the shell script using:

```
./burnintest.sh.
```

If you encounter errors about missing Qt5 libraries, proceed to the next step.

5. Resolve Missing Dependencies: To resolve the missing Qt5 libraries, install a package that includes these libraries, such as qt5-assistant , by running:

```
sudo apt install qt5
-
assistant
```

6. Run the Test as Superuser: After installing the necessary libraries, rerun the test script with superuser privileges:

```
sudo ./burnintest.sh
```

7. Configure Test Parameters: In the BurnInTest UI, go to the Configuration tab to select the specific tests you want to run. Adjust the parameters according to your testing needs.

8. Start the Test: Click the green “Play” button to begin the test. The system will run under high load for an extended period.

9. Review the Results: After the test completes, check the “Burn In Results” tab for any errors or issues that were detected. These results will help you assess the system’s resilience and identify any potential weaknesses under prolonged stress.
