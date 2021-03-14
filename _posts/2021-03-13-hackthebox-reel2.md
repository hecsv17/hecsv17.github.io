Hello friends!
Today I'm going to take around a special machine on hackthebox plateform: Reel2
It was a hard machine as the statistiques show!
First we start as usual with running <emb>nmap</emb> to see open ports and services.
<emb>nmap -sV -T4 -oA nmap 10.10.10.210</emb>
The result shows a bunch of ports: 80, 443, 8080, 600x.
We will start by ports 80 and 443, which only show a default windows IIS server default page. In this case we will launch a directory/files brute force to see if can find some juicy entries.  
