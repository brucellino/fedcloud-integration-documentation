OpenStack
`````````

Integration with FedCloud requires a *working OpenStack installation* as a pre-requirement (see http://docs.openstack.org/ for details). This manual provides information on how to set up a Resource Centre providing cloud resources in the EGI infrastructure, using **OpenStack Ocata**. EGI expect the following OpenStack services to be available and accessible from outside the site:

* Keystone

* Nova

* Cinder

* Glance

* Neutron

* Swift (if providing Object Storage)

FedCloud components are distributed through `CMD (Cloud Middleware Distribution) <https://wiki.egi.eu/wiki/EGI_Cloud_Middleware_Distribution>`_ or docker container images available in dockerhub. These docker containers come pre-packaged and ready to use in the EGI FedCloud Appliance so you do not need to install any extra components on your site but just run a VM and configure it approprietely to interact with your services.

The integration is performed by a set of EGI components that interact with the OpenStack services APIs:

.. image:: /_static/OpenStackSite.png

* **cASO** collects accounting data from OpenStack and uses **SSM** to send the records to the central accounting database on the EGI Accounting service (APEL)

* **cloud-info-provider** registers the RC configuration and description through the EGI Information System to facilitate service discovery

* **cloudkeeper** (and **cloudkeeper-os**) synchronises with `EGI AppDB <https://appdb.egi.eu/browse/cloud>`_  so new or updated images can be provided by the RC to user communities (VO).

Not all EGI components need to share the same credentials. They are individually configured, you can use different credentials and permissions if desired.

Authentication by EGI users into your system is performed by configuring the native OpenID Connect support of Keystone. Support for legacy VOs using VOMS requires the installation of the **Keystone-VOMS Authorization plugin** to  allow users with a valid VOMS proxy to obtain tokes to access your OpenStack deployment.

Optionally, **ooi (OpenStack OCCI Interface)** translates between OpenStack API and OCCI.


.. TODO
   PORTS?


EGI AAI
:::::::

OpenID Connect Support
''''''''''''''''''''''

The integration of OpenStack service providers into the EGI CheckIn is a two-step process:

#. Test integration with the development instance of EGI CheckIn. This will allow you to check complete the complete functionality of the system without affecting  the production CheckIn service.

#. Once the integration is working correctly, register your provider with the production instance of EGI CheckIn to allow members of the EGI User Community to access your service.

Registration into CheckIn development instance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Before your service can use the EGI CheckIn OIDC Provider for user login, you must set up a client at https://aai-dev.egi.eu/oidc/manage/#admin/clients in order to obtain OAuth 2.0 credentials and register one or more redirect URIs.

Make sure that you fill in the following options:

* Main tab:

    * Set redirect URL to ``https://<your keystone endpoint>/v3/auth/OS-FEDERATION/websso/oidc/redirect``. Recent versions of OpenStack may deploy Keystone at ``/identity/``, be sure to include that in the ``<your keystone endpoint>`` part of the URL if needed.

* Access tab:

    * Enable '''authorization code''' in the grant type
    * Enable '''Introspection Allow calls to the Introspection Endpoint?'''

Once done, you will get a client id and client secret. Save them for the following steps

Keystone setup
~~~~~~~~~~~~~~

Pre-requisites
""""""""""""""

#. Keystone must run as a WSGI application behind an HTTP server (Apache is used in this documentation, but any server should be possible if it has OpenID connect/OAuth2.0 support). Keystone project has deprecated eventlet, so you should be already running Keystone in such way.

#. Keystone must be run with SSL

#. You need to install [https://github.com/pingidentity/mod_auth_openidc mod_auth_openidc] for adding support for OpenID Connect to Apache.

Apache Configuration
~~~~~~~~~~~~~~~~~~~~

Include this configuration on the Apache config for the virtual host of your Keystone service, using the client id and secret obtained above:

::

    OIDCResponseType "code"
    OIDCClaimPrefix "OIDC-"
    OIDCClaimDelimiter ;
    OIDCScope "openid"
    OIDCProviderMetadataURL https://aai-dev.egi.eu/oidc/.well-known/openid-configuration
    OIDCClientID <client id>
    OIDCClientSecret <client secret>
    OIDCCryptoPassphrase <some crypto pass phrase>
    OIDCRedirectURI https://<your keystone endpoint>/v3/auth/OS-FEDERATION/websso/oidc/redirect

    # OAuth for CLI access
    OIDCOAuthIntrospectionEndpoint  https://aai-dev.egi.eu/oidc/introspect
    OIDCOAuthClientID <client id>
    OIDCOAuthClientSecret <client secret>

    <Location ~ "/v3/auth/OS-FEDERATION/websso/oidc">
            AuthType  openid-connect
            Require   valid-user
    </Location>

    <Location ~ "/v3/OS-FEDERATION/identity_providers/egi.eu/protocols/oidc/auth">
            Authtype oauth20
            Require   valid-user
    </Location>

Be sure to enable the mod_auth_oidc module in Apache, in Ubuntu:

::

    sudo a2enmod auth_openidc

Keystone Configuration
~~~~~~~~~~~~~~~~~~~~~~

Configure your ``keystone.conf`` to include in the ``[auth]`` section ``oidc`` in the list of authentication methods and the ``keystone.auth.plugins.mapped.Mapped`` class for its implementation:


::

    [auth]

    # This may change in your installation, add oidc to the list of the methods you support
    methods = password, token, oidc

    # OIDC is basically mapped auth method
    oidc = keystone.auth.plugins.mapped.Mapped

Add a ``[oidc]`` section as follows:

::

    [oidc]
    # this is the attribute in the Keystone environment that will define the identity provider
    remote_id_attribute = HTTP_OIDC_ISS

Add your horizon host as trusted dashboard to the ``[federation]`` section:

::

    [federation]
    trusted_dashboard = https://<your horizon>/dashboard/auth/websso/

Finally copy the default template for managing the tokens in horizon to ``/etc/keystone/sso_callback_template.html``. This template can be found in keystone git repo at ``https://github.com/openstack/keystone/blob/master/etc/sso_callback_template.html``

::

    curl -L https://raw.githubusercontent.com/openstack/keystone/master/etc/sso_callback_template.html \
        > /etc/keystone/sso_callback_template.html

Now restart your Apache (and Keystone if running in uwsgi) so you can configure the Keystone Federation support.

Keystone Federation Support
~~~~~~~~~~~~~~~~~~~~~~~~~~~

First, create a new  ``egi.eu`` identity provider with remote id ``https://aai-dev.egi.eu/oidc/``:

::

    $ openstack identity provider create --remote-id https://aai-dev.egi.eu/oidc/ egi.eu
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | None                             |
    | domain_id   | 1cac7817dafb4740a249cc9ca6b14ea5 |
    | enabled     | True                             |
    | id          | egi.eu                           |
    | remote_ids  | https://aai-dev.egi.eu/oidc/     |
    +-------------+----------------------------------+

Create a group for users coming from EGI CheckIn, usual configuration is to have one group per VO you want to support.

::

    $ openstack group create ops
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description |                                  |
    | domain_id   | default                          |
    | id          | 89cf5b6708354094942d9d16f0f29f8f |
    | name        | ops                              |
    +-------------+----------------------------------+

Add that group to the desired local project:

::

    $ openstack role add member --group ops --project ops

Define a mapping of users from EGI CheckIn to the group just created and restrict with the ``OIDC-edu_person_entitlements`` the VOs you want to support for that group. Substitute the group id and the allowed entitlements for the adequate values for your deployment:

::

    $ cat mapping.egi.json
    [
        {
            "local": [
                {
                    "user": {
                "name": "{0}"
            },
                    "group": {
                        "id": "89cf5b6708354094942d9d16f0f29f8f"
                    }
                }
            ],
            "remote": [
                {
                    "type": "HTTP_OIDC_SUB"
                },
                {
                    "type": "HTTP_OIDC_ISS",
                    "any_one_of": [
                        "https://aai-dev.egi.eu/oidc/"
                    ]
                },
                {
                    "type": "OIDC-edu_person_entitlements",
                    "regex": true,
                    "any_one_of": [
                        "^urn:mace:egi.eu:.*:vm_operator@ops$"
                    ]
                }
            ]
        }
    ]

More recent versions of Keystone allow for more elaborated mapping, but this configuration should work for Mitaka and onwards

Create the mapping in Keystone:

::

    $ openstack mapping create --rules mapping.egi.json egi-mapping
    +-------+----------------------------------------------------------------------------------------------------------------------------------+
    | Field | Value                                                                                                                            |
    +-------+----------------------------------------------------------------------------------------------------------------------------------+
    | id    | egi-mapping                                                                                                                      |
    | rules | [{u'remote': [{u'type': u'HTTP_OIDC_SUB'}, {u'type': u'HTTP_OIDC_ISS', u'any_one_of': [u'https://aai-dev.egi.eu/oidc/']},        |
    |       | {u'regex': True, u'type': u'OIDC-edu_person_entitlements', u'any_one_of': [u'^urn:mace:egi.eu:.*:ops:vm_operator@egi.eu$']}],    |
    |       | u'local': [{u'group': {u'id': u'89cf5b6708354094942d9d16f0f29f8f'}, u'user': {u'name': u'{0}'}}]}]                               |
    +-------+----------------------------------------------------------------------------------------------------------------------------------+

Finally, create the federated protocol with the identity provider and mapping created before:

::

    $ openstack federation protocol create --identity-provider egi.eu --mapping egi-mapping oidc
    +-------------------+-------------+
    | Field             | Value       |
    +-------------------+-------------+
    | id                | oidc        |
    | identity_provider | egi.eu      |
    | mapping           | egi-mapping |
    +-------------------+-------------+

Keystone is now ready to accept EGI CheckIn credentials.

Horizon Configuration
~~~~~~~~~~~~~~~~~~~~~


Edit your local_settings.py to include the following values:

::

    # Enables keystone web single-sign-on if set to True.
    WEBSSO_ENABLED = True

    # Allow users to choose between local Keystone credentials or login
    # with EGI CheckIn
    WEBSSO_CHOICES = (
        ("credentials", _("Keystone Credentials")),
        ("oidc", _("EGI CheckIn")),
    )

Once horizon is restarted you will be able to choose "EGI CheckIn" for login.

CLI Access
~~~~~~~~~~


The `OpenStack Client <https://docs.openstack.org/developer/python-openstackclient/>`_ has built-in support for using OpenID Connect Access Tokens to authenticate. You first need to get a valid token from EGI CheckIn (e.g. from https://aai-dev.egi.eu/fedcloud/) and then use it in a command like:

::

    $ openstack --os-auth-url https://<your keystone endpoint>/v3 \
                --os-auth-type v3oidcaccesstoken --os-protocol oidc \
                --os-identity-provider egi.eu \
                --os-access-token <your access token> \
                token issue
    +---------+---------------------------------------------------------------------------------------+
    | Field   | Value                                                                                 |
    +---------+---------------------------------------------------------------------------------------+
    | expires | 2017-05-23T11:24:31+0000                                                              |
    | id      | gAAAAABZJA3fbKX....nEMAPi-IsFOCkU9QWGTISYElzYJsI3z0SJGs7QsTJv4aJQq0JDJUBz6uE85SqXDj3  |
    | user_id | 020864ea9415413f9d706f6b473dbeba                                                      |
    +---------+---------------------------------------------------------------------------------------+

Additional VOs
~~~~~~~~~~~~~~

Configuration can include as many mappings as needed in the json file. Users will be members of all the groups matching the remote part of the mapping. For example this file has 2 mappings, one for members of ``ops`` and another for members of ``fedcloud.egi.eu``:

::

    [
        {
            "local": [
                {
                    "user": {
                "name": "{0}"
            },
                    "group": {
                        "id": "66df3a7a0c6248cba8b729de7b042639"
                    }
                }
            ],
            "remote": [
                {
                    "type": "HTTP_OIDC_SUB"
                },
                {
                    "type": "HTTP_OIDC_ISS",
                    "any_one_of": [
                        "https://aai-dev.egi.eu/oidc/"
                    ]
                },
                {
                    "type": "OIDC-edu_person_entitlements",
                    "regex": true,
                    "any_one_of": [
                        "^urn:mace:egi.eu:.*:vm_operator@ops$"

                    ]
                }
            ]
        },
        {
            "local": [
                {
                    "user": {
                "name": "{0}"
            },
                    "group": {
                        "id": "e1c04284718f4e19bb0516e5534a24e8"
                    }
                }
            ],
            "remote": [
                {
                    "type": "HTTP_OIDC_SUB"
                },
                {
                    "type": "HTTP_OIDC_ISS",
                    "any_one_of": [
                        "https://aai-dev.egi.eu/oidc/"
                    ]
                },
                {
                    "type": "OIDC-edu_person_entitlements",
                    "regex": true,
                    "any_one_of": [
                        "^urn:mace:egi.eu:.*:vm_operator@fedcloud.egi.eu$"
                    ]
                }
            ]
        }
    ]

Moving to EGI CheckIn production instance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**TBC**

.. Register your Service Provider with the production instance of EGI CheckIn to allow members of the EGI User Community to access your service. This requires that your service meets all the [[AAI_guide_for_SPs#Services eligible for integration|eligibility criteria]] and that integration has been thoroughly tested during Step 1.


VOMS Support
''''''''''''

**VOMS Support using Keystone-VOMS is no longer supported from OpenStack Queens onwards**

Support for authenticating users with X.509 certificates with VOMS extensions is achieved with Keystone-VOMS extension. Documentation is available at https://keystone-voms.readthedocs.io/

Notes:

* **You need a host certificate from a recognised CA for your keystone server**.

* Take into account that using keystone-voms plugin will **enforce the use of https for your Keystone service**, you will need to update your URLs in the configuration of your services if your current installation is not using https:

  * you will probably need to include your CA to your system's CA bundle to avoid certificate validation issues: Check the `Federated Cloud OpenStack Client guide <https://wiki.egi.eu/wiki/Federated_Cloud_APIs_and_SDKs#CA_CertificatesCheck>`_ on how to do it.
  * replace http with https in ``auth_[protocol|uri|url]`` and ``auth_[host|uri|url]`` in the nova, cinder, glance and neutron config files (``/etc/nova/nova.conf``, ``/etc/nova/api-paste.ini``, ``/etc/neutron/neutron.conf``, ``/etc/neutron/api-paste.ini``, ``/etc/neutron/metadata_agent.ini``, ``/etc/cinder/cinder.conf``, ``/etc/cinder/api-paste.ini``, ``/etc/glance/glance-api.conf``, ``/etc/glance/glance-registry.conf``, ``/etc/glance/glance-cache.conf``) and any other service that needs to check keystone tokens.

  * Update the URLs of the services directly in the database:

::

    mysql> use keystone;
    mysql> update endpoint set url="https://<keystone-host>:5000/v2.0" where url="http://<keystone-host>:5000/v2.0";
    mysql> update endpoint set url="https://<keystone-host>:35357/v2.0" where url="http://<keystone-host>:35357/v2.0";

* Most sites should enable the ``autocreate_users`` option in the ``[voms]`` section of `Keystone-VOMS configuration <https://keystone-voms.readthedocs.org/en/latest/configuration.html>`_. This will enable new users to be automatically created in your local keystone the first time they login into your site.

* if (and only if) you need to configure the Per-User Subproxy (PUSP) feature, please follow the `specific guide <https://wiki.egi.eu/wiki/Long-tail_of_science_-_information_for_providers#Instructions_for_OpenStack_providers>`_.

EGI VM Management (optional)
::::::::::::::::::::::::::::

Follow the `installation and configuration manual of ooi <http://ooi.readthedocs.org/en/stable/index.html>`_.

.. TODO: packages?

Once the OCCI interface is installed, you should register it on your installation (adapt the region and URL to your deployment), e.g.:

::

    $ openstack service create --name occi --description "OCCI Interface" occi
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | OCCI Interface                   |
    | enabled     | True                             |
    | id          | 6dfd6a56c9a6456b84e8c86038e58f56 |
    | name        | occi                             |
    | type        | occi                             |
    +-------------+----------------------------------+

    $ openstack endpoint create --region RegionOne occi --publicurl http://172.16.4.70:8787/occi1.1

    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |           OCCI service           |
    |      id     | 8e6de5d0d7624584bed6bec9bef7c9e0 |
    |     name    |             occi_api             |
    |     type    |               occi               |
    +-------------+----------------------------------+


Other components
::::::::::::::::

There are two options to install the remaining Accounting, Information System and Image Management components:

* Using the EGI FedCloud Appliance (recommended), which uses docker containers to bundle an OpenStack deployment of the corresponding services

* Using individual components.

Follow the guides below according to your preferences

FedCloud Appliance
''''''''''''''''''

The EGI FedCloud Appliance is available at `AppDB <https://appdb.egi.eu/store/vappliance/fedcloud.integration.appliance.openstack>`_ as an OVA file. You can easily extract the VMDK disk by untaring and optionally converting it to your preferred format with qemu-img:

::

    # get image and extract VMDK
    curl https://cephrgw01.ifca.es:8080/swift/v1/egi_endorsed_vas/FedCloud-Appliance.Ubuntu.16.04-2017.08.09.ova | \
           tar x FedCloud-Appliance.Ubuntu.16.04-2017.08.09-disk001.vmdk
    # convert to qcow2
    qemu-img convert -O qcow2 FedCloud-Appliance.Ubuntu.16.04-2017.08.09-disk001.vmdk fedcloud-appliance.qcow2

The VM running at your OpenStack must:

* Be accessible via public IP with port 2170 open for external connections.

* Have a host certificate to send the accounting information to the accounting repository. DN of the host certificate must be registered in GOCDB service type eu.egi.cloud.accounting. The host certificate and key in PEM format are expected in /etc/grid-security/hostcert.pem and /etc/grid-security/hostkey.pem respectively.

* Have enough disk space for handling the VM image replication (~ 100GB for `fedcloud.egi.eu` VO). By default these are stored at /image_data. You can mount a volume at that location.

EGI Accounting
~~~~~~~~~~~~~~

There are two different processes handling the accounting integration:

* cASO, which connects to the OpenStack deployment to get the usage information, and,

* ssmsend, which sends that usage information to the central EGI accounting repository.

They are run by cron every hour (cASO) and every six hours (ssmsend).

`cASO configuration <http://caso.readthedocs.org/en/latest/configuration.html>`_ is stored at  ``/etc/caso/caso.conf``. Most default values are ok, but you must set:

* ``site_name`` (line 12)

* ``projects`` (line 20)

* credentials to access the accounting data (lines 28-47, more options also available). Check the `cASO documentation <http://caso.readthedocs.org/en/latest/configuration.html#openstack-configuration>`_ for the expected permissions of the user configured here.

The cron job will use the voms mapping file at ``/etc/voms.json``.

cASO will write records to ``/var/spool/apel`` from where ssmsend will take them.

SSM configuration is available at ``/etc/apel``. Defaults should be ok for most cases. The cron file uses ``/etc/grid-security`` for the CAs and the host certificate and private keys (``/etc/grid-security/hostcert.pem`` and ``/etc/grid-security/hostkey.pem``).

Running the services
""""""""""""""""""""

Both caso and ssmsend are run via cron scripts. They are located at ``/etc/cron.d/caso`` and ``/etc/crond.d/ssmsend`` respectively. For convenience there are also two scripts ``/usr/loca/bin/caso-extract.sh`` and ``/usr/local/bin/ssm-send.sh`` that run the docker container with the proper volumes.

EGI Information System
~~~~~~~~~~~~~~~~~~~~~~

Information discovery provides a real-time view about the actual images and flavors available at the OpenStack for the federation users. It has two components:

* Resource-Level BDII: which queries the OpenStack deployment to get the information to publish

* Site-Level BDII: gathers information from several resource-level BDIIs (in this case only 1) and makes it publicly available for the EGI information system.

Resource-level BDII
"""""""""""""""""""

This is provided by container ``egifedcloud/cloudbdii``. You need to configure:

* ``/etc/cloud-info-provider/openstack.rc``, with the credentials to query your OpenStack. The user configured just needs to be able to access the lists of images and flavors.

* ``/etc/cloud-info-provider/openstack.yaml``, this file includes the static information of your deployment. Make sure to set the ``SITE-NAME`` as defined in GOCDB.

Site-level BDII
"""""""""""""""

The ``egifedcloud/sitebdii`` container runs this process. Configuration files:

* `/etc/sitebdii/glite-info-site-defaults.conf`. Set here the name of your site (as defined in GOCDB) and the public hostname where the appliance will be available.

* `/etc/sitebdii/site.cfg`. Include here basic information on your site.

Running the services
""""""""""""""""""""

There is a `bdii.service` unit for systemd available in the appliance. This leverages docker-compose for running the containers. You can start the service with:

::

    systemctl start bdii

Check the status with:

::

    systemctl status bdii

And stop with:

::

    systemctl stop bdii

You should be able to get the BDII information with an LDAP client, e.g.:

::

    ldapsearch -x -p 2170 -h <yourVM.hostname.domain.com> -b o=glue

EGI VM Image Management
~~~~~~~~~~~~~~~~~~~~~~~

The appliance provides VMI replication with cloudkeeper. Every 4 hours, the appliance will perform the following actions:

* download the configured lists in ``/etc/cloudkeeper/image-lists.conf`` and verify its signature

* check any changes in the lists and download new images

* synchronise this information to the configured glance endpoint

cloudkeeper has two components:

* fronted dealing the with image lists and downloading the needed images

* backend dealing with your glance catalogue

First you need to configure and start the backend. Edit ``/etc/cloudkeeper/cloudkeeper-os.conf`` and add the authentication parameters from line 117 to 136.
Then add as many image lists (one per line) as you would like to subscribe to ``/etc/cloudkeeper/image-lists.conf``. Use URLs with your AppDB token for authentication.

Running the services
""""""""""""""""""""

cloudkeeper-os should run permanently, there is a ``cloudkeeper-os.service`` for systemd in the appliance. Manage as usual:

::

    systemctl <start|stop|status> cloudkeeper-os

cloudkeeper core is run every 4 hours with a cron script.

Individual Components
'''''''''''''''''''''

EGI Accounting
~~~~~~~~~~~~~~


Every cloud RC should publish utilization data to the EGI accounting database. You will need to install **cASO**, a pluggable extractor of Cloud Accounting Usage Records from OpenStack.

Documentation on how to install and configure cASO is available at https://caso.readthedocs.org/en/latest/

In order to send the records to the accounting database, you will also need to configure **SSM**, whose documentation can be found at https://github.com/apel/ssm


EGI Information System
~~~~~~~~~~~~~~~~~~~~~~

Sites must publish information to EGI information system which is based on BDII. The BDII can be installed easily directly from the distribution repository, the package is usually named "bdii".

There is a common cloud information provider for all cloud management frameworks that collects the information from the used CMF and send them to the aforementioned BDII. It can be installed on the same machine as the BDII or on another machine. The installation and configuration guide for the cloud information provider can be found in the following `Fedclouds BDII instructions <https://wiki.egi.eu/wiki/HOWTO15>`_.

EGI VM Image Management
~~~~~~~~~~~~~~~~~~~~~~~

.. TODO: Where are the docs?

Post-installation
:::::::::::::::::

After the installation of all the needed components, it is recommended to set the following policies on Nova to avoid users accessing other users resources:

::

    sed -i 's|"admin_or_owner":  "is_admin:True or project_id:%(project_id)s",|"admin_or_owner":  "is_admin:True or project_id:%(project_id)s",\n    "admin_or_user":  "is_admin:True or user_id:%(user_id)s",|g' /etc/nova/policy.json
    sed -i 's|"default": "rule:admin_or_owner",|"default": "rule:admin_or_user",|g' /etc/nova/policy.json
    sed -i 's|"compute:get_all": "",|"compute:get": "rule:admin_or_owner",\n    "compute:get_all": "",|g' /etc/nova/policy.json