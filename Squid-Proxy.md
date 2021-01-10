# Proxy Authenticated Account with Squid Proxy

- Instead of login everytime with domain or another account, just simply login once and proxy it with Squid to use your proxy.
- You should first install squid for this.
```bash
sudo yum update -y
sudo yum install squid -y
sudo yum clean all
```

# Squid Configuration

- After installation done, **/etc/squid/squid.conf** should be edited. 
- The important sections to be edited are **acl(access control list)**  and **proxy cache** sections.
1.  **acl** section defines wihich **IPs** or **IP Ranges** you want to let use your authenticated proxy.
```bash
# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src xxx.xxx.xxx.xxx     # You can change with IPs or IP ranges
acl localnet src xxx.xxx.xxx.xxx/XX  # You can change with IPs or IP ranges

acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
acl Safe_ports port 70          # gopher
acl Safe_ports port 210         # wais

.................
.................
```
2. After that authenticated proxy can be entered via **cache_peer**. Like below, put **cache_peer** rules after   **INSERT YOUR OWN RULE(S)** section. With that you would not need to think if you break default rules or not. Edit **username:password** and save it. 
```bash
.................

#
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
#
cache_peer 10.45.146.93 parent 8080 0 default no-query login=username:password
cache_peer 10.45.145.93 parent 8080 0 default no-query login=username:password
never_direct allow all

.................
```
- After these parts **Squid Proxy** can be enabled and opened.
```bash
sudo systemctl start squid
sudo systemctl enable squid
```
