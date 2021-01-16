
### Purpose:

Configure a basic Windows 10 Pro, v20H2 machine to support the
following:
 1. Multiple virtual environments using either Microsoft's Hyper-V or VMWare's  
 VMWare version 16 or above (the author tested this process using VMWare 16.1)
 2. Installing Docker **within** a virtual machine _as well as_ on the host  
 itself. Getting them both to work is somewhat tricky, the integration isn't  
 firmly established with Microsoft just yet.
 3. Install Windows Subsystem for Linux both on the host and within a virtual  
 machine.
 4. Demonstrate some of the interoperability between the Windows "host" and the  
 WSL sub-system (calling one type of program, e.g. a Linux command, from within  
 the Windows enviornment, e.g. Powershell and vice-versa).
