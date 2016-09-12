# SOPHOS Chef Cockbook

## SG (Security Gateway - UTM 9)

The SG recipies use the UTM 9 REST API to automate provisioning of the UTM.
Basic requirement is, that the basic setup has been done before (will create
the admin account etc.).

To find out how to configure the UTM 9 use the `confd-watch.plx -v` command.
It will indicate create objects (o+), changed objects (oc), changed nodes (nc)
and object deletions (o-). Use the output to generate the recipes for your UTM
instances:

Note: in most cases you can omit empty values like: `Empty SCALAR`,
`Empty ARRAY`, etc. Use `true` and `false` for `1` and `0` if its a status field.

### Configuration

To configure the utm, that will be configured with the chef recipies, one can
use the following attributes:

#### URL

The sophos sg url is the url to the UTM to configure, embed the username
and password into the url. Be sure to use the `https` scheme, the correct
port (`4444`) and api path (`/api`).

    default['sophos']['sg']['url'] = 'https://admin:passwd@example.org:4444/api'

#### Fingerprint (for SSL without valid certificate chain)

In case you use the default self signed certificate of your UTM, and you don't
want to install an official / or install it to your trusted ones, you can choose
to use **Public key fingerprinting**. To find out the fingerprint of your utm
use openssl:

    openssl s_client -connect <your-ip-or-dns>:4444 < /dev/null 2>/dev/null |\
        openssl x509 -fingerprint -noout -in /dev/stdin

_(If the command above doesn't return a fingerprint, your openssl might be to old)_

Then use the fingerprint in your configuration:

    default['sophos']['sg']['fingerprint'] = 'FF:00:80:BE:89:3E:CA:7C:A4:C3:03:AF:1F:18:99:7D:75:D2:69:01'

### Examples

Here are some examples that where created using the output of the
`confd-watch.plx -v` command. If further assitance on the datanmodel is needed,
consult the REST API at `https://<your host>/api/` on your utm and inspect
the different objects and nodes, the `POST` formular gives good inside in
possible values.

#### WEB Filtering

**Enable Application Control:**

`confd-watch.plx -v` output:

    s  1  caught USR1 signal(s)
    vc 27 28  data version change detected at Fri Sep  9 07:43:30 2016
    nc afc->status 1

Chef translation:

    sophos_sg_node 'afc.status' do
      value true
    end

**Enable WEB filtering:**

`confd-watch.plx -v` output:

    oc REF_DefaultHTTPProfile http profile status  changed
       status = 1

Chef translation:

    sophos_sg_object 'http/profile/REF_DefaultHTTPProfile' do
      attributes status: true
      action :change
    end

**Creating a domain regex (lowbird.com) to filter and start blocking
unappropriate content:**

`confd-watch.plx -v`:

    o+ REF_HttDomLowbirdcom http domain_regex  created
       restrict_regex = 1
       include_subdomains = 1
       domain = [ lowbird.com ]
       comment = Empty SCALAR
       mode = Domain
       regexps = Empty ARRAY
       name = lowbird.com
    oc REF_DefaultHTTPCFFAction http cff_action sp_categories,url_blacklist  changed
       sp_categories = [ REF_CriminalActivities, REF_Drugs, REF_ExtremisticSites, REF_GamesGambles ]
       url_blacklist = [ REF_HttDomLowbirdcom ]

Chef:

    sophos_sg_object 'http/domain_regex/REF_HttDomLowbirdcom' do
      attributes restrict_regex: true,
                 include_subdomains: true,
                 domain: [ 'lowbird.com' ],
                 mode: 'Domain',
                 name: 'lowbird.com'
      action :create
    end

    sophos_sg_object 'http/cff_action/REF_DefaultHTTPCFFAction' do
      attributes sp_categories: [ 'REF_CriminalActivities',
                                  'REF_Drugs',
                                  'REF_ExtremisticSites',
                                  'REF_GamesGambles' ],
                 url_blacklist: [ 'REF_HttDomLowbirdcom' ]
      action :change
    end

#### Packetfilter

Allow HTTPS, SMTP, SSH from internal to mailserver:

    sophos_sg_object 'network/host/REF_NetHosMailseInDe' do
      attributes name: 'Mailserver in DE',
                 address: '5.35.240.160'
      action :create
    end

    sophos_sg_object 'packetfilter/packetfilter/REF_AllowMailAccess' do
      auto_insert_to_node 'packetfilter.rules'
      attributes sources: ['REF_DefaultInternalNetwork'],
                 services: ['REF_MeigLDviNK',
                            'REF_SWVaJaLGTT',
                            'REF_nUyAxjnNLV'],
                 destinations: ['REF_NetHosMailseInDe'],
                 name: 'HTTPS from Internal to Mail',
                 action: 'accept',
                 log: true,
                 status: true
      action :create
    end


Allow developer network to access internal network:

    sophos_sg_object 'network/network/REF_NetDevelopers' do
      attributes name: 'Network of the developers',
                 address: '1.2.3.0',
                 netmask: 24
      action :create
    end

    sophos_sg_object 'packetfilter/packetfilter/REF_PacAllowAnyFromDevelopers' do
      auto_insert_to_node 'packetfilter.rules'
      attributes sources: ['REF_NetDevelopers'],
                 services: ['REF_ServiceAny'],
                 destinations: ['REF_DefaultInternalNetwork'],
                 name: 'Any From Dev To UTM internal',
                 action: 'accept',
                 log: true,
                 status: true
      action :create
    end

#### Advanced Threat Protection

Enable Advanced Threat Protection:

    sophos_sg_node 'aptp.status' do
      value true
    end

#### Masquerading

Enable masquerading from the internal network on the wan interface:

    sophos_sg_object 'packetfilter/masq/REF_MasqInternToWEB' do
      auto_insert_to_node 'masq.rules'
      attributes source: 'REF_DefaultInternalNetwork',
                 name: 'from Internal (Network) to WEB',
                 source_nat_interface: 'REF_IntEthExternaWan',
                 status: true
      action :create
    end

#### DNAT

Redirect HTTP traffic fom Any to the Public Address to the Webserver:

    sophos_sg_object 'network/host/REF_NetHosWebserver' do
      attributes name: 'Webserver',
                 address: '10.106.194.42'
      action :create
    end

    sophos_sg_object 'network/host/REF_NetHosPubliAddress' do
      attributes name: 'Public Address',
                 address: '1.2.3.4'
      action :create
    end

    sophos_sg_object 'packetfilter/nat/REF_PacNatHttpFromAny' do
      auto_insert_to_node 'nat.rules'
      attributes source: 'REF_NetworkAny',
                 service: 'REF_zbCXCkAONs',
                 name: 'HTTP from Any to public address',
                 source_nat_interface: 'REF_IntEthExternaWan',
                 destination: 'REF_NetHosPubliAddress',
                 destination_nat_address: 'REF_NetHosWebserver',
                 auto_pfrule: true,
                 mode: 'dnat',
                 status: true
      action :create
    end

#### WAF

Enable webserver protection for host heise.de (only http) for domain
frontend.utm-chef.com:

    sophos_sg_object 'network/dns_host/REF_NetDnsHeise' do
      attributes name: 'Heise',
                 hostname: 'heise.de'
      action :create
    end

    sophos_sg_object 'reverse_proxy/backend/REF_RevBacHeise' do
      attributes name: 'Heise Backend',
                 host: 'REF_NetDnsHeise',
                 path: '/',
                 port: 80,
                 status: true
      action :create
    end

    sophos_sg_object 'reverse_proxy/location/REF_RevLoc' do
      attributes backend: ['REF_RevBacHeise'],
                 name: '/',
                 stickysession_id: 'ROUTEID',
                 path: '/',
                 be_path: '',
                 allowed_networks: ['REF_NetworkAny']
      action :create
    end

    sophos_sg_object 'reverse_proxy/frontend/REF_RevFroFrontWebse' do
      attributes htmlrewrite_cookies: true,
                 status: true,
                 profile: '',
                 certificate: '',
                 allowed_networks: ['REF_NetworkAny'],
                 lbmethod: 'bybusyness',
                 domain: ['frontend.utm-chef.com'],
                 disable_compression: false,
                 add_content_type_header: true,
                 address: 'REF_DefaultInternalAddress',
                 preservehost: false,
                 locations: ['REF_RevLoc'],
                 name: 'frontend webserver',
                 htmlrewrite: false,
                 port: 80,
                 xheaders: false,
                 type: 'http',
                 implicitredirect: true
      action :create
    end
