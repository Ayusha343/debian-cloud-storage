# You want to create a personal FTP global server using Cloudflare tunneling, but face the issue that the Cloudflare tunnel URL changes every time you restart the service, making the URL non-permanent.
To solve this, you need a **cheap or free domain name** to have a stable, permanent URL that you can control.
I suggested using:
A **cheap domain name** from providers like Hostinger, BigRock, or free options like Freenom for niche TLDs.
To manage your dynamic IP and keep your domain pointing to your changing IP, use a **dynamic DNS service called dyn.addr.tools**.
It uses a secret-based subdomain on dyn.addr.tools you link via a CNAME record to your domain.
You update the IP address associated with this subdomain via simple HTTP API calls whenever your IP changes.
This way, your custom domain/subdomain always resolves to your current IP, providing a permanent URL.
This setup integrates well with Cloudflare tunneling by giving you a stable domain that points to your dynamic home or server IP.
You asked if this approach will work, and I confirmed it is a practical and effective solution widely used to handle dynamic IP addresses.  
  
Yes, your proposed approach is indeed practical and effective for creating a stable, permanent URL for your personal FTP global server using Cloudflare tunneling despite dynamic IP changes.

Here’s a summary of why this works well:

- Acquiring a cheap or free domain name from providers like Hostinger, BigRock, or Freenom gives you full control over a permanent domain or subdomain.

- Using a dynamic DNS service like dyn.addr.tools handles your changing IP by allowing you to update the DNS record of a secret-based subdomain through simple API calls. This ensures that the subdomain always points to your current IP.

- Linking your custom domain or subdomain to this dynamic DNS subdomain via a CNAME record means your domain will automatically resolve to the correct IP address without manual reconfiguration.

- With Cloudflare tunnel serving your FTP server, you then have a stable full URL (your custom domain) that clients can use continuously, even if your IP or the tunnel URL changes.

This method is commonly used to overcome the dynamic IP problem for home servers and services exposed to the internet. It neatly combines domain ownership, dynamic DNS updating, and Cloudflare's proxying to maintain accessibility and permanence in your URL.

So yes, your strategy is sound and suited perfectly for your goal. Just ensure you configure the dynamic DNS updates correctly and set the proper DNS records in your domain registrar’s panel to link everything together.  
---  


# DYN.ADDR.TOOLS.
Domain Name System
DYN.ADDR.TOOLS.
NAME
dyn.addr.tools - simple dynamic DNS for your own domains, no account required
SYNOPSIS

*$* curl 'https://dyn.addr.tools/?secret=1SuperSecret&ip=self'
OK


*$* curl -d 'secret=1SuperSecret' -d 'ip=192.0.2.1' [https://dyn.addr.tools](https://dyn.addr.tools)
OK


*$* sha224=$(echo -n '1SuperSecret' | shasum -a 224 | cut -c 1-56)
*$* dig +short $sha224.dyn.addr.tools
192.0.2.1

DESCRIPTION
dyn.addr.tools is a dynamic DNS service with a simple HTTP interface meant to point subdomains (via CNAME) to non-static IP addresses.
USAGE
Let ipaddr be an IPv4 or IPv6 address, secret be a strong password, and sha224 be the SHA-224 hash of secret.
To update the domain `sha224.dyn.addr.tools` to resolve to ipaddr, make a GET, POST, or PUT request to [`https://dyn.addr.tools`](https://dyn.addr.tools) with `secret=secret` and `ip=ipaddr` specified as URL query parameters or, alternatively for POST and PUT requests, as form values. Responds with body `OK` and status code `200` on success.
ipaddr may be the word "self" to use the requester's public IPv4 or IPv6 address. Use host ipv4.dyn.addr.tools or ipv6.dyn.addr.tools to force IPv4 or IPv6.
A GET, POST, or PUT to [`https://dyn.addr.tools`](https://dyn.addr.tools) with only `secret=secret` specified responds with body `sha224.dyn.addr.tools` and makes no update.
A DELETE to [`https://dyn.addr.tools`](https://dyn.addr.tools) with `secret=secret` specified removes both IPv4 and IPv6 addresses. Responds with status code `204` on successful removal of all addresses (which may be zero).
`sha224.dyn.addr.tools` is meant to be the target of a CNAME at your own subdomain.
At least one update must be made every 90 days to remain active. IPv4 and IPv6 updates are tracked independently.
Remember to properly encode your secret value in your requests if it contains special characters. See curl's "--data-urlencode" option.
EXAMPLE
Say you want to keep home.example.com updated with your public IPv4 address.
First, pick a strong password. We'll use "1SuperSecret", but you shouldn't.
Add a CNAME record to point home.example.com to the subdomain of dyn.addr.tools named by calculating the SHA-224 hash of "1SuperSecret". This should look similar to:
Name:
home.example.com
Type:
CNAME
Target:
1d23d5e1a9a689668e8510aef992aa358cb54992d0c4327842a1416f.dyn.addr.tools
Update the subdomain to resolve to your public IPv4 address:

*$* curl 'https://ipv4.dyn.addr.tools/?secret=1SuperSecret&ip=self'

Use a program, cron job, etc., to make a similar update request when the IP changes and at least once every 90 days.
For example, you can use pfSense to keep your IP updated by adding a new Dynamic DNS Client (under Services) with these settings:
Service Type:
Custom
Update URL:
[https://ipv4.dyn.addr.tools/?secret=1SuperSecret&ip=self](https://ipv4.dyn.addr.tools/?secret=1SuperSecret&ip=self)
Result Match:
OK
Max Cache Age:
90
ACCEPTABLE USE
Users of this service will not engage in distributing malware, abusively operating botnets, phishing, piracy, trademark or copyright infringement, fraudulent or deceptive practices, counterfeiting, or otherwise engaging in activity contrary to applicable law. Such behavior may lead to the deactivation of the subdomain.
Please send reports of unacceptable use to .
SEE ALSO
[addr.tools](https://addr.tools/)

go through this  
  
The documentation you provided for dyn.addr.tools explains how this dynamic DNS service works and how to integrate it with your domain to keep your DNS pointing to your frequently changing IP address. Here’s a concise breakdown of the key points and instructions:

### What dyn.addr.tools Does
- It provides dynamic DNS by assigning a special subdomain under `dyn.addr.tools` that resolves to your current IP address.
- This subdomain is derived by hashing a secret password using SHA-224, creating a stable but secret subdomain like `<sha224hash>.dyn.addr.tools`.
- You use HTTP requests (GET, POST, PUT) with your secret and IP address to update the IP behind this subdomain.
- Your own domain/subdomain can point to `<sha224hash>.dyn.addr.tools` via a CNAME record.

### How to Use It
1. **Choose a strong secret password**, for example, `"1SuperSecret"` (use your own secret).
2. **Calculate the SHA-224 hash** of the secret, which becomes the subdomain under `dyn.addr.tools`.
   - For `"1SuperSecret"` the hash is:  
     `1d23d5e1a9a689668e8510aef992aa358cb54992d0c4327842a1416f` (example from doc).
3. **Add a CNAME record** in your domain DNS settings pointing your custom subdomain (e.g., `home.example.com`) to  
   `1d23d5e1a9a689668e8510aef992aa358cb54992d0c4327842a1416f.dyn.addr.tools`.
4. **Update your IP address** on dyn.addr.tools:
   - Use this HTTP API call to update it with your current IP (automate via script or router):  
     `curl 'https://ipv4.dyn.addr.tools/?secret=1SuperSecret&ip=self'`  
     (`ip=self` means the service will detect your public IPv4 automatically.)
5. **Keep updating at least once every 90 days** to keep the record active.
6. Your clients can then connect using your custom domain `home.example.com`, which will resolve to your current IP via dyn.addr.tools.

### Additional Notes
- You can update IPv4 or IPv6 separately by using `ipv4.dyn.addr.tools` or `ipv6.dyn.addr.tools`.
- Requests without IP only return the current subdomain but do not update IP.
- DELETE requests with your secret can remove the DNS entries.
- Proper URL and secret encoding is necessary if your secret has special characters.

### Summary
This approach provides a simple, free, and account-less dynamic DNS solution for your own domains, perfectly matching your desire for a cheap or free stable domain linked to your changing IP for your Cloudflare tunneled FTP server.

If you setup such a CNAME and run regular IP updates like above, your custom domain will remain a permanent and stable URL for your server.

Let me know if you want help making a script or cron job to automate these updates!  
---  


# so, if i  understand it right, using rhis dyn,addr.tools i can get a domain, which i can use with cloudfare tunneling to get a permanent url even after i restart the service  
  
Yes, you understand it correctly.

Using dyn.addr.tools, you get a stable subdomain (derived from your secret’s hash) that always points to your current IP, even if it changes. By creating a CNAME from your own custom domain or subdomain to that dyn.addr.tools subdomain, you effectively map your domain to your dynamic IP.

When you use Cloudflare tunneling for your server, you can configure it to use this custom domain. Since your domain always resolves correctly to your current IP via dyn.addr.tools, your URL remains permanent and stable—even after restarting the tunnel service or when your IP changes.

So, dyn.addr.tools provides the dynamic DNS part that keeps your domain pointing to your live IP, and Cloudflare tunnel handles the secure exposure, giving you a permanent URL overall.  
---  


# my isp provides dynamic ip and i cant port forward. will it work too? are there sny downside/  
  
If your ISP provides a dynamic IP and you cannot do port forwarding, using Cloudflare tunneling combined with dyn.addr.tools can still work, but there are some important considerations and potential downsides:

### Will It Work?
- **Yes, with Cloudflare tunneling:**  
  Cloudflare Tunnel (formerly Argo Tunnel) creates an outbound connection from your server to Cloudflare’s network, so it does not require port forwarding on your router. This means even if your ISP blocks incoming ports or you cannot forward ports, your server can still be accessed securely through Cloudflare’s network.

- **dyn.addr.tools helps with DNS:**  
  If your public IP changes, dyn.addr.tools will keep your domain/subdomain updated so it resolves to the correct IP, but since Cloudflare Tunnel creates an outbound connection, this IP dynamicity does not affect access to your services behind the tunnel.

### Downsides and Limitations:
- **Not a direct IP-based access:**  
  Since Cloudflare tunnel runs over an outbound connection, you don’t access your server’s IP directly—Cloudflare acts as a proxy. All traffic goes through Cloudflare’s edge network, which means you depend on Cloudflare’s availability and infrastructure.

- **Performance and latency:**  
  Because of the additional routing through Cloudflare, latency might be slightly higher compared to direct port forwarding.

- **Cost for advanced Cloudflare features:**  
  While the basic Cloudflare Tunnel service is free, some advanced features or higher usage might require a paid Cloudflare plan.

- **No external raw IP access:**  
  You won’t have direct access via raw IP or traditional FTP on your public IP; access must always go through the tunnel and domain.

- **Needs software running:**  
  The Cloudflare tunnel client must be running continuously on your server/device to maintain the connection.

### Summary
With no port forwarding and a dynamic IP, your best practical solution is Cloudflare Tunnel combined with dyn.addr.tools for dynamic DNS. It overcomes the port forwarding limitation by not relying on inbound connections, and dyn.addr.tools ensures your custom domain always resolves correctly.

This setup is very effective, widely used, and reliable, but it does depend on Cloudflare’s infrastructure and introduces slight overhead due to proxying.

If you want, I can guide you through setting up this entire environment step-by-step.