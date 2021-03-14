Hello friends!
Today I'm going to take you around a special machine on hackthebox platform: Reel2
<img src="/src/reel2.png">

It was a hard machine as the statistiques show!
First we start as usual with running <emb>nmap</emb> to see open ports and services.
<span style="color:blue"><emb>nmap -sV -T4 -oA nmap 10.10.10.210</emb></span>
The result shows a bunch of ports: 80, 443, 8080, 600x.


We will start by ports 80 and 443, which only show a default windows IIS server default page. In this case we will launch a directory/files brute force to see if can find some juicy entries.

<span style="color:blue"><emb>dirsearch.py -e php,html -u http://10.10.10.210/ -t 50 -w wordlist.txt</emb></span>


<span style="color:blue"><emb>dirsearch.py -e php,html -u https://10.10.10.210/ -t 50 -w wordlist.txt</emb></span>


The results show the existing of exchange server and the web posrtal is accessible through https://10.10.10.210/owa (we can guess from the fisrt look that the exchange version is 2010)

Next step is to build username and password lists to brute force the OWA access. At this point we can use large wordlists but it may take an eternity to finish the attack.

The next open port is 8080 which exposes a social media website called "wallstant". We create a new account and start to look for some vulnerablities on that site. As by now, we will use this site to generate wordlists for username and password.


The following command will generate a worldlist using CEWL program (-m command is used for minimum word length, -d command used depth to spider):


<span style="color:blue"><emb>cewl -w passlist.txt -m 2 -d 4 -H Cookie:PHPSESSID=q7lmf0h6m81u4propu0r1s4reh --proxy_host 127.0.0.1 --proxy_port 8080 http://10.10.10.210:8080/</emb></span>
To generate a wordlist for usernames we make a curl to search endpoint which contains names of blog users.


<span style="color:blue"><emb>curl -v --cookie "PHPSESSID=q7lmf0h6m81u4propu0r1s4reh" http://10.10.10.210:8080/search > site.txt</emb></span>
The following will create a list from the output of curl command:


<span style="color:blue"><emb>html2dic site.txt >> list.txt</emb></span>
