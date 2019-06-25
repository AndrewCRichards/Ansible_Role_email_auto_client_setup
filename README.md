Ansible role: email auto client setup
=====================================

This is an Ansible role to set up files and Apache vhost file(s) to serve autodiscover XML
files over HTTPS (and using PHP) for email client software using the Microsoft autodiscover
standard, and to serve autoconfig XML files over HTTP for email client software using the
Thunderbird autoconfig standard. Both of these standards have the aim of discovering the
server settings needed to configure an email client automatically given a user's email address.

The use of free Let's Encrypt SSL certificates is assumed, you can override the provided
template files with your own local versions if you're using a different provider for your
SSL certificates (or of course if you wish to make other adjustments to them).

If you're using a different web server you may still find parts of this role useful
to generate the autodiscover, autoconfig pages even if you can't use the vhost part.

Possibly more information, esp. about autodiscover / autoconfig standards for setting
up email client software at [my own site](https://free.acrconsulting.co.uk/); I'm also
available for [commercial support](https://www.acrconsulting.co.uk/) on this and other
issues around Ansible and email servers.

RFC 6186
--------

In case you're not aware of [RFC 6186](https://tools.ietf.org/html/rfc6186), my advice
is: Setup RFC 6186 records for your domain before bothering with autodiscover/autoconfig
from this role: This is a simpler and more modern way of configuring
autodiscover/autoconfig via DNS ``SRV`` records, I suggest setting that up first
before using this role, which is really to provide completeness so that [esp. legacy]
email clients that aren't RFC-6186-aware can still find their configuration. If you're
after a quick-fix for autodiscover/autoconfig, I suggest that you should setup RFC-6186
first - for example in your domain's zone file (from the RFC),

    _imap._tcp       SRV  0 1 143 imap.example.com.
    _pop3._tcp       SRV 10 1 110 pop3.example.com.
    _submission._tcp SRV  0 1 587 mail.example.com.

If you still need autodiscover/autoconfig after setting up RFC 6186 records for your
domain, read on...

Requirements
------------

Ansible 2.2 or higher.

For Microsoft autodiscover: PHP (the XML template needs to generate the &lt;``LoginName``&gt; field
based on supplied POST data).

The role assumes that you're using Let's Encrypt SSL certificates; you can override the vhost template
with your own if you have SSL certificates from a different provider.

Apache is also assumed, but again, the role shouldn't be difficult to adjust for a different web server.

Except for a simple single-domain setup, some DNS setup may be needed. A possible DNS setup might look like this,

 - Thunderbird:       ``autoconfig.example.org CNAME--> some.central.domain``
 - Microsoft: ``_autodiscover._tcp.example.org  SRV --> some.central.domain``

in named/BIND syntax for a given zone/domain, that would be,

    autoconfig		IN	CNAME		some.central.domain
    _autodiscover._tcp	IN	SRV	0 0 443	some.central.domain

(This role sets up the XML files, vhost files for the web server on ``some.central.domain`` in the above example)

Role Variables
--------------

Have one or more domains specified in ``email_auto_client_setup_vhosts`` (see Example Setup below),

Other variables that can be set are shown here with their default values,
for most scenarios you're unlikely to need to override these,

    email_auto_client_setup_web_server_package_name: apache2
    email_auto_client_setup_web_server_modules_etc: True
    email_auto_client_setup_web_server_modules_to_enable:
        - ssl
        - alias
    email_auto_client_setup_template_locations:
        - "{{ inventory_dir }}/templates/{{ inventory_hostname }}/{{ role_name }}"
        - "{{ inventory_dir }}/templates/{{ role_name }}"
        - "{{ role_path }}/templates/local"
        - "{{ role_path }}/templates"

    email_auto_client_setup_defaults:
         setup_MS: True
         setup_TB: True
         setup_vhost: False

Notice ``email_auto_client_setup_template_locations``: Override the role's supplied example
templates by placing your customised templates in one of the first 2 directories shown.

Dependencies
------------

None in terms of Ansible roles, but read the above notes re Apache, SSL, DNS

Example Setup
----------------

You may choose to put your site's settings for this role in the Ansible ``hosts`` file;
alternatively you may find it clearer to put the settings in a file
``host_vars/``*machine_name*``/email_auto_client_setup``. Put customised templates in
``templates/``*machine_name*``/email_auto_client_setup`` or ``templates/email_auto_client_setup``.

Your settings will need to include the domain(s) you're setting up as virtualhosts,

    email_auto_client_setup_vhosts :
       - domain_name      : cnametarget.example.net
         email_settings:
           provider_domain  : cnametarget.example.net
           pop3_server      : pickup.example.org
           imap_server      : pickup.example.org
           submission_server: smtp.example.org
         server_admin     : postmaster@cnametarget.example.net
         document_root    : "/some/path/cnametarget.example.net/html"
         setup_MS         : True
         setup_TB         : True
         setup_vhost      : True

Tip: If you're using an SSL connection including TLS, STARTTLS etc, make
sure the specified server(s) response(s) include a certificate for the
server name(s) you give here - so in the above example ``pickup.example.org``
would respond with a SSL certificate for ``pickup.example.org`` for POP3,
IMAP connections; similarly ``smtp.example.org`` would respond with a SSL
certificate for ``smtp.example.org`` for mail submission connections. Smaller
organisations are likely to have the same server handle POP3, IMAP and mail
submission in which case all 3 would have the same name, ``something.example.org``

See 'Role Variables' above for other variables that can be set if needed.

Testing autoconfig
------------------

Once you've setup a host for autoconfig with this role, you can test the result by issuing command-line requests such as,

    curl http://autoconfig_target_domain/.well-known/autoconfig/mail/config-v1.1.xml
    curl http://autoconfig_target_domain/mail/config-v1.1.xml

where ``autoconfig_target_domain`` refers to the target domain of a DNS ``CNAME`` record
for ``_autoconfig._tcp.``*your_domain*, or for simpler setups (without ``SRV`` record)
may just be *your_domain* and/or ``autoconfig.``*your_domain*.

Testing autodiscover
--------------------

Once you've setup a host for autodiscover with this role, you can test the result by issuing a command-line POST request such as,

    curl -XPOST -d @req.xml --header "Content-Type:text/xml" https://autodiscover_target_domain/autodiscover/autodiscover.xml

where ``autodiscover_target_domain`` refers to the target domain of a DNS ``SRV`` record
for ``_autodiscover._tcp.``*your_domain*, or for simpler setups (without ``SRV`` record)
may just be *your_domain* and/or ``autodiscover.``*your_domain*.

where ``req.xml`` looks like this,

    <?xml version="1.0" encoding="utf-8"?>
    <Autodiscover xmlns="http://schemas.microsoft.com/exchange/autodiscover/outlook/requestschema/2006">
      <Request>
        <AcceptableResponseSchema>http://schemas.microsoft.com/exchange/autodiscover/outlook/responseschema/2006a</AcceptableResponseSchema>
        <EMailAddress>someone@your.domain.here</EMailAddress>
      </Request>
    </Autodiscover>

esp. helpful on getting this right was [this site](https://github.com/jeroenj/autodiscover).

License
-------

GPLv3 - see files in 'meta' directory

Author Information
------------------

Andrew Richards: https://acrconsulting.co.uk/
