# Custom domains

Your DNS settings should include a CNAME record pointing the subdomain of your
choice (`subdomain.example.org`) to your seedbox's hostname
(`example-server.seedhost.eu`). If you wish to have the root domain
`example.org` host the web server, include an A record for `example.org`
pointing to `example-server.seedhost.eu`'s IP address. (Though SeedHost does not
advertise its dedicated hosts as having static IPs, they do for the sake of some
default internal configuration.)

If your DNS provider has advanced tools such as proxies with URL rewriting or
request header modification, these can make some configurations easier (or at
least more aesthetically pleasing). I use a free Cloudflare proxy for this
purpose (as well as for my own DNS provider).

Recall that external clients connect to the custom web server (Apache) on
SeedHost servers via proxy from the nginx server; if desired, Cloudflare could
bypass nginx completely by redirecting traffic from `www.example.org` to
`http://example-server.seedhost.eu:8000`. However, the Apache server does not
serve https by default, so this would be an insecure method of bypassing nginx.
Further customization of this sort of configuration (such as setting up SSL for
Apache) is left as an exercise for the reader, but could theoretically be useful
to bypass objectionable nginx settings that can't be changed.

Using the standard SeedHost configuration, any requests we wish to be served by
the Apache server must start with the path `/username/`. To remove this
redundancy for a custom domain, [Cloudflare Transform
Rules](https://developers.cloudflare.com/rules/transform/) can rewrite any URL
that doesn't include the leading `/username` path to one that does, so that
`https://www.example.org` and `https://www.example.org/username` serve the same
content. This may, however, have SEO implications, which can then be solved by
additionally creating a redirect from `https://www.example.org/username` to
`https://www.example.org/`. If not done carefully, this can lead to an
unresolving loop. Cloudflare documentation suggests the URL rewrite occurs
before the redirect, so we must check whether the "raw" request (prior to
rewrite) starts with `/username` before redirecting it to remove this bit, then
to be silently re-written on the next request by the rewrite rule. Actual
behavior without checking the "raw" request seems to work as intended. Yes, it's
all silly, but so are the search engines.

Additionally, Cloudflare proxies normally prevent non-HTTP/HTTPS traffic going
to the web server. If your `example.org` host name resolves to a Cloudflare
proxy, tools like SSH are unlikely to connect directly to `example.org`. There
are various solutions to this:

- use the IP address or `example-server.seedhost.eu` rather than `example.org`
- use mechanisms for the tool to alias `example.org` to one of the above. E.g.,
  for ssh, use a `~/.ssh/config` `Host` entry
- turn off the Cloudflare proxy for the bare domain `example.org` (which
  therefore also turns off any of the configured rules for requests to that
  address)
- create an alternate subdomain such as `ssh.example.org` with the Cloudflare
  proxy off and use that rather than `example.org`

Finally, one helpful rule to set in Cloudflare for custom domains is a Transform Rule to
Modify Request Header, setting X-Forwarded-Host for all incoming requests. This
should be a dynamic setting with value expression `http.host`. This header is
not typically set by Cloudflare by default, and some applications on the Apache
server may expect it when redirecting incoming requests.
