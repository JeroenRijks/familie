SSL encryption

Zed, Stath and I worked on enabling SSL encryption in a Rails API, and I thought I'd share what I've learnt, and what things I still need to learn :)

When users use our frontend, it sends requests from their browsers, to our API, via two things:
A content delivery network (Cloudflare), which essentially increases response time by caching (remembering things that other people have asked the API). Anything that Cloudflare doesn't remember, it forwards to the API, like normal
A load balancer, which receives traffic from Cloudflare, and sends it to the least busy Rails API server

That means that overall, we've got three connections, which should all use SSL encryption, for maximum security
`client browser -> Cloudflare` - This is encrypted by a cloudflare certificate
`Cloudflare -> AWS load balancer` - This is encrypted by AWS
`Load balancer -> Rails API` - This wasn't encrypted, but we're working on it

The fact that the `load balancer -> API` connection isn't encrypted isn't terrifying, because we own the load balancer, so we can be pretty sure that it's forwarding all traffic to the API. However, there's no harm in setting encryption up, so that's what we tried.

#### Certificate
Before encryption begins, the client must ensure that it's talking to the correct server. The server proves its authenticity by sending a certificate, who's authenticity can be vouched for by a certificate authority. We have chosen to use self-signed certificates, to avoid operational overhead, generating a certificate and key, and passing these into the API as environment variables.

#### Implementation question
If the `SSL_KEY` and `SSL_CERTIFICATE` environment variables are populated when starting a Rails pod, it'll use the values when starting Puma, and otherwise, it will leave the LB -> API connection unencrypted.

```
// config/puma.rb

ssl_key = ENV['SSL_KEY']
ssl_certificate = ENV['SSL_CERTIFICATE']
ip_address = ENV['IP_ADDRESS'] || 'localhost'
port = '80'

if ssl_key.present? && ssl_certificate.present?
  bind "ssl://#{ip_address}:#{port}?key=#{ssl_key}&cert=#{ssl_certificate}"
else
  bind "tcp://#{ip_address}:#{port}"
end
```

Certificate expiration
I'm using self-signed certificates that last 365 days, and I'm worried that we might forget to update them, if there's no error message. Any tips on avoiding certificate expirations in prod, other than using a calendar? For example, do you recommend making a cert that's valid for 10 years?