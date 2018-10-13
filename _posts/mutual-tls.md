# Mutual TLS Authentication using private CA Chains

I've been asked to write down some experiences with private CA chains
and mutual TLS authentication I'm currently operating. So here we go.

As a warning - this document is not intended as series of easily copy-pasted
openssl cli invocations. Articles like that are just one web search 
away, but they are not going to help you in this case.

Setting up private TLS and especially mutual TLS authentication is 
more like networking and less than using a REST API. With a REST API,
or many other problem spaces, you can do a lot through trial, error
and iteration. This is not a bad practice in itself - often it is faster
to just try a couple of things and advance from there. 

However, this approach requires quick iteration and good error reporting.
If you have good errors to work with, you can just deduce what's wrong
and change it. Networking, VPN tunnels and TLS authentication tends to
have rather terrible error reporting. If you misconfigure a dropping
firewall, you get zero response. If you misconfigure IPSec tunnels, you're 
lucky to get something like <CHILD_SA mismatch>, which has a couple
possible reasons. Usually your traffic gets dropped in kernel space without
any further mention. Or, if you're running vault with client TLS authentication,
you just get an HTTP response code 401 or 403 and that's it. Good luck
figuring out which of the half a dozen different reasons you're running into.

In my experience, in this kind of problem space you have to know exactly why
your solution will work before you even try it. Otherwise, the troubleshooting
will be very, very painful. On top of that, libraries handling TLS certificates
tend to be obscure and terse. And some of them are even picky and confusing on
top of that - looking at you, `java.security`. 

That is why I am going to spend quite a bit of time working out and talking
about the fundamentals at work. This article should bring you to the point of
"Ok how do I do this specific thing with my stack?" or "Ok I fully grok what this
system wants regarding certificates, let's do that". 

Over the rest of this article:

 - I'll elaborate my motivations for implementing mutual authentication from a
   security and business perspective
 - I'll talk about my mindset while dealing with these systems
 - I'll elaborate on the mechanics, terms and situations dealing with mutual
   TLS authentication and CA chains
 - And finally I'll elaborate some on the automation we're currently
   deploying based on hashicorp vault. 

# Motivation

Mutual authentication is a bit of an unintuitive beast initially. For example,
if you're used to a relational database - MySQL, PostgreSQL - you give it a 
password and it gives you data. Why should an application server verify 
it's talking to the right database? The wrong database shouldn't have that 
password anyway, else we would have a security breach already, because 
database passwords have been leaked.  This is entirely true - for this case. 
The application server unconditionally trusts the relational database. If
the database outputs bogus data, the application server outputs bogus data
and then we get customer tickets.

However, the rabbit hole grows deeper in our operational infrastructure in
several ways. In our infrastructure, we have the configuration management 
request passwords for database users from vault servers, and re-configures
the database users accordingly. This is much more interesting all of a 
sudden.

It would be an obvious disaster if an unauthenticated client could request
secrets from the secret management. It would almost render the secrets to
not be secrets.

However, things get more interesting if you consider that
request to a vault server are redirected to a malicious server under the
control of an attacker. Without server-side authentication, the server
could tell our configuration management to set all passwords in our database
to "hunter2" and the configuration management would happily execute this
command. With proper server-side authentication, the configuration management
immediately aborts upon connection without considering anything else
the server might send. In this situation, our TLS setup a client would 
not even disclose it's own public key or signature to a compromised server. 

This is an improvement over simple client-side authentication. If you
could MITM a simple password authentication, you could just get the client 
passwords. This does not happen in a mutual TLS setup. 

- TODO: logstash example (GDPR)
