# Lab answers: HTTP servers, SSH tunnels, SOCKS proxies

## Part 1

HTTP port: **33561** (chosen by the OS with `python3 -m http.server 0 --bind 0.0.0.0 --directory www`).

- curl to `127.0.0.1:33561` on the server: **worked** (HTTP/1.0 200 OK and my HTML).
- curl to `212.34.250.14:33561` (server's real address) on the server: **worked** (200 OK).
- curl from my local machine (Mac) to `212.34.250.14:33561`: **worked**, printed my HTML. The server log showed the request coming from my public address 5.77.x.x.
- Browser on my local machine to `http://212.34.250.14:33561/`: **worked**. The browser marked the page "Not secure" because it is plain HTTP; I continued to the HTTP site.

`netstat -anp | grep -i python` showed python listening on `0.0.0.0:33561`.

## Part 2

After restarting with `--bind 127.0.0.1`:

- curl to `127.0.0.1:33561` on the server: **worked** (200 OK).
- curl to `212.34.250.14:33561` on the server: **failed** ("Failed to connect ... Could not connect to server").
- curl from my Mac to `212.34.250.14:33561`: **failed** (connection refused).

`ss -ltn` showed: `LISTEN 0 5 127.0.0.1:33561 0.0.0.0:*`

- **What does 0.0.0.0 mean?** The server listens on all IPv4 interfaces of the machine, so connections to any of its IPv4 addresses (loopback and the public 212.34.250.14) are accepted.
- **What changes with 127.0.0.1?** The server listens only on the loopback interface, so only programs running on the same machine can connect. Connections to the public address are refused.
- **Does 127.0.0.1 mean only my Unix user can connect?** No. Loopback limits *which machine* can connect, not *which user*. Every user logged in to the shared course server can connect to `127.0.0.1:33561`. To restrict by user you would need something else (e.g. a Unix socket with file permissions, or authentication).

## Part 3

- Before the tunnel: `http://127.0.0.1:8080/` on my Mac **failed** (nothing listening).
- During the tunnel (`ssh -N -L 127.0.0.1:8080:127.0.0.1:33561 serzh_harutyunyan@212.34.250.14`): the page **loaded** and showed my page. The Python log showed the requests coming from 127.0.0.1.
- After stopping the tunnel with Ctrl+C: it **failed** again.

- **On which machine is port 8080 listening?** On my local machine (the Mac); the ssh client opens it.
- **On which machine is the Python HTTP port listening?** On the course server, on 127.0.0.1:33561.
- **What does the second 127.0.0.1 refer to?** It is resolved by the SSH server, so it is the course server's own loopback address.
- **Why can the browser reach a loopback-only service?** The browser connects to ssh on local port 8080; ssh sends the data through the encrypted SSH connection; sshd on the server then opens a connection to 127.0.0.1:33561. To Python the connection comes from the server itself, so the loopback-only bind does not block it.

Screenshot: `evidence/http-tunnel.png`.

## Part 4

Firefox was not installed on my Mac, so I used a separate Chrome instance started with
`--proxy-server=socks5://127.0.0.1:1080` (Chrome sends DNS lookups through a SOCKS5 proxy, like Firefox's "Proxy DNS when using SOCKS v5").

- Without SOCKS, ip.me showed my own address: **5.77.x.x** (IPv4).
- With SOCKS (`ssh -N -D 127.0.0.1:1080 serzh_harutyunyan@212.34.250.14`), ip.me showed **2001:470:22c2:250:7468:6568:6c6c:1**, organization "Hurricane Electric IPv6 tunnel broker". This is the course server's outgoing IPv6 address: the server reached ip.me over IPv6, so it is not the same as the IPv4 address on its interface (212.34.250.14).
- With the proxy off (normal browser), my own address 5.77.x.x came back.
- curl test: `curl https://api.ipify.org` gave **5.77.x.x**; `curl --socks5-hostname 127.0.0.1:1080 https://api.ipify.org` gave **212.34.250.14** (ipify answers over IPv4, so here the server's IPv4 address is shown).

- **Is port 1080 listening on my machine or the server?** On my local machine; the ssh client opens it.
- **Does a SOCKS proxy in the browser change the traffic of every application?** No. Only the browser configured to use it (and programs explicitly told to use it, like curl with `--socks5-hostname`) go through the proxy. Other applications still connect directly.
- **Encryption:** the connection between my computer and the SSH server is encrypted by SSH. The connection from the SSH server to the destination website is not encrypted by SSH; it is only encrypted if the site itself uses HTTPS/TLS.
- **Difference from a VPN like WireGuard:** a VPN creates a virtual network interface and routes IP packets (TCP, UDP, everything) for the whole system, usually all applications, and needs admin/root rights to set up. A SOCKS proxy over SSH works per application and per TCP connection, needs no root and creates no interface.
- **Difference from Yggdrasil:** Yggdrasil is an encrypted overlay mesh network: every node gets its own IPv6 address and can reach other nodes directly, with routing across many peers. An SSH tunnel is just a single point-to-point relay through one server; my machine does not get a new address on any network.

Screenshot: `evidence/socks-ip.png`.
