:tocdepth: 1

.. sectnum::

Abstract
========

We plan to use COmanage as the identity management platform for the Rubin Science Platform.
This document describes how to configure COmanage for that purpose and proposes a design for the necessary integration services to retrieve user metadata, group membership, and group metadata.

Configuration
=============

Add username to enrollment flow
-------------------------------

#. Edit "Self Signup With Approval" enrollment flow
#. Edit its enrollment attributes
#. Add username with a suitable description.
   Allow the user to change it during enrollment.
   Set the type of the field to CO Person, Identifier, UID.
   Mark as required.

This does not work for the "Invite a collaborator" enrollment flow, since the person creating the invite is prompted for the username.
We probably won't need that flow.
If we do, we'll have to find a way to modify it to defer the username choice to the person who was invited.

Configure LDAP provisioning target
----------------------------------

#. Go to Configuration -> Provisioning Targets and configure Primary LDAP
#. Go down to the attribute configuration
#. Enable ``displayName``, disable ``givenName``, and set it to Official
#. Change ``uid`` to use the UID identifier
#. Enable ``groupOfNames`` objectclass
#. Enable ``hasMember`` in the ``eduMember`` objectclass
#. Enable ``voPerson`` and set ``voPersonID`` to the LSST Registry ID and ``voPersonSoRID`` to System of Record ID
#. Save and then Reprovision All to update existing records

Dashboard
---------

COmanage comes with a bunch of default components that we probably don't want to use (announcement feeds, forums, etc.).
We will want to edit the default dashboard to remove those widges and replace them with widges for group management and personal identity management (if there are any applicable ones).

API
===

REST API
--------

Only the `REST v1 API <https://spaces.at.internet2.edu/display/COmanage/REST+API+v1>`__ is currently available.
The base URL is the hostname of the COmanage registry service with ``/registry`` appended.

LDAP
----

To make LDAP queries, use commands like:

.. code-block:: console

   $ ldapsearch -LLL -H ldaps://ldap-test.cilogon.org \
                -D 'uid=readonly_user,ou=system,o=LSST,o=CO,dc=lsst,dc=org' \
                -x -w PASSWORD -b 'ou=people,o=LSST,o=CO,dc=lsst,dc=org'

The password is in 1Password under the hostname of the COmanage registry.

An example user::

    dn: voPersonID=LSST100006,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    sn: Allbery
    cn: Russ Allbery
    objectClass: person
    objectClass: organizationalPerson
    objectClass: inetOrgPerson
    objectClass: eduMember
    objectClass: voPerson
    displayName: Russ Allbery
    mail: rra@lsst.org
    uid: rra
    isMemberOf: CO:members:all
    isMemberOf: CO:members:active
    isMemberOf: CO:admins
    isMemberOf: science-platform-idf-dev
    voPersonID: LSST100006
    voPersonSoRID: http://cilogon.org/serverA/users/31388556

The ``voPersonID`` without the ``LSST`` prefix should be usable as a numeric UID.

An example group::

    dn: cn=science-platform-idf-dev,ou=groups,o=LSST,o=CO,dc=lsst,dc=org
    cn: science-platform-idf-dev
    member: voPersonID=LSST100006,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    member: voPersonID=LSST100007,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    objectClass: groupOfNames
    objectClass: eduMember
    hasMember: rra
    hasMember: thoron

Note that the group entry in LDAP doesn't contain numeric GID information.
See :ref:`Numeric GIDs <gid>` for more details.

.. _gid:

Numeric GIDs
------------

Getting numeric GIDs into the LDAP entries for each group isn't well-supported by COmanage.
The LDAP connector does not have an option to add arbitrary group identifiers to the group LDAP entry.
There are a few possible options.

Grouper
"""""""

A look at what capabilities Grouper has to assign GIDs and expose them via an API or via LDAP is upcoming, pending addition of Grouper to the test environment.

Group REST API
""""""""""""""

Arbitrary identifiers can be added to groups, so a group can be configured with an auto-incrementing unique identifier in the same way that we do for users, using a base number of 200000 instead of 100000 to keep the UIDs and GIDs distinct (allowing the UID to be used as the GID of the primary group).
Although that identifier isn't exposed in LDAP, it can be read via the COmanage REST API using a URL such as::

    https://<registry-url>/registry/identifiers.json?cogroupid=7

The group ID can be obtained from the ``/registry/co_groups.json`` route, searching on a specific ``coid``.
Middleware running on the Rubin Science Platform could cache the GID information for every group, refresh it periodically, and query for the GID of a new group when seen.

voPosixGroup
""""""""""""

Another option is to enable ``voPosixGroup`` and generate group IDs that way.
However, that process is somewhat complex.

COmanage Registry has the generic notion of a `Cluster <https://spaces.at.internet2.edu/display/COmanage/Clusters>`__.
A Cluster is used to represent a CO Person's accounts with a given application or service.

Cluster functionality is implemented by Cluster Plugins.
Right now there is one Cluster Plugin that comes out of the box with COmanage, the `UnixCluster plugin <https://spaces.at.internet2.edu/display/COmanage/Unix+Cluster+Plugin>`__.

The UnixCluster plugin is configured with a "GID Type."
From the documentation we read "When a CO Group is mapped to a Unix Cluster Group, the CO Group Identifier of this type will be used as the group's numeric ID."
CO Person can then have a UnixCluster account that has associated with it a UnixCluster Group, and the group will have a GID identifier.

To have the information about the UnixCluster and the UnixCluster Group provisioned into LDAP using the ``voPosixAccount`` objectClass, you need to define a `CO Service <https://spaces.at.internet2.edu/display/COmanage/Registry+Services>`__ for the UnixCluster.
In that configuration you need to specify a "short label", which will become value for an LDAP attribute option.
Since the ``voPosixAccount`` objectClass attributes are multi-valued, you can represent multiple "clusters," and they are distinguised by using that LDAP attribute option value.
For example::

    dn: voPersonID=LSST100000,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    sn: KORANDA
    cn: SCOTT KORANDA
    objectClass: person
    objectClass: organizationalPerson
    objectClass: inetOrgPerson
    objectClass: eduMember
    objectClass: voPerson
    objectClass: voPosixAccount
    givenName: SCOTT
    mail: SKORANDA@CS.WISC.EDU
    uid: http://cilogon.org/server/users/2604273
    isMemberOf: CO:members:all
    isMemberOf: CO:members:active
    isMemberOf: scott.koranda UnixCluster Group
    voPersonID: LSST100000
    voPosixAccountUidNumber;scope-primary: 1000000
    voPosixAccountGidNumber;scope-primary: 1000000
    voPosixAccountHomeDirectory;scope-primary: /home/scott.koranda

This reflects a CO Service for the UnixAccount using the short label "primary."
With a second UnixCluster and CO Service with short label "slac" to represent an account at SLAC, then I would have additionally::

    voPosixAccountGidNumber;scope-slac: 1000001

UnixCluster object and UnixCluster Group objects and all the identifiers are usually established during an enrollment flow.

Custom development
""""""""""""""""""

We could enhance (or pay someone to enhance) the LDAP Provisioning Plugin to allow us to express an additional object class in the group tree in LDAP, containing a numeric GID identifier.

Integration
===========

We will need to write the following services to integrate with COmanage.

User information API
--------------------

Gafaelfawr is currently temporarily recording and returning metadata about a user, such as full name, numeric UID, and group information, based on the CILogon assertions.
(See SQR-049_ for more details.)
One goal of adopting COmanage as the identity management system is to drop this information from Gafaelfawr and retrieve it directly from COmanage.

.. _SQR-049: https://sqr-049.lsst.io/

This will require a new internal API service in the Rubin Science Platform.
Services can authenticate it using a Gafaelfawr token and retrieve metadata about the user.
This should include:

* Full name
* Primary email address
* Numeric UID
* Group membership with numeric GIDs for each group

To reduce latency and load on the COmanage API, this service should cache those results for some to-be-determined period of time.
We should consider having a mechanism for the user to invalidate the cache (such as on logout).

This corresponds primarily to the ``/auth/api/v1/user-info`` route specified in SQR-049_, with the addition of email.

This API service may also need to support integration with GitHub and with the OpenID Connect and LDAP provider used at the base and summit so that we can remove the remaining user metadata support in Gafaelfawr.
Alternately, we could use COmanage for those environments as well, but that would likely not meet the off-line requirements for the summit environment, and there is merit in the flexibility to quickly stand up a Rubin Science Platform deployment using GitHub as the identity management system.

It appears the preferred interface in COmanage to pull this type of user metadata is LDAP.

Full name should always be ``displayName`` and we should not use the other LDAP attributes that attempt to parse a name into components.
They do not internationalize well.
Unfortunately, the COmanage sign-on flow still asks for users to enter their name in components.

User onboarding API
-------------------

The "Self Signup With Approval" flow seems to be the closest fit for our requirements.
To initiate that flow, we send the user to a specific URL at the COmanage registry.
We can initiate that flow from the landing page or from Gafaelfawr if we detect that the user is authenticated but not enrolled in COmanage.

It's possible to then configure a return URL to which the user goes after enrollment is complete, but that's probably not that useful when we're using an approval flow.
We will need to customize the email messages and web pages presented as part of the approval flow.

When a new user first accesses the Rubin Science Platform, we will need to route them through the onboarding flow, and then may need to make additional changes to their record via the COmanage API such as adding them to groups.
This can be integrated with the onboarding service described in SQR-052_.
This service would have a privileged API token for the Rubin Science Platform COmanage environment.

.. _SQR-052: https://sqr-052.lsst.io/

The web pages shown during this onboarding flow are controlled by the style information in the `lsst-registry-landing <https://github.com/cilogon/lsst-registry-landing>`__ project on GitHub.

Currently, user onboarding has a bug: After choosing their name, email, and username, the user is sent an email message to confirm that they have control over that email address.
The link in the mail message has a one-time code in it, and confirms the email address when followed.
However, sites with anti-virus integrated with their email system (such as AURA) often pre-fetch all URLs seen in email addresses.
Since no authentication or confirmation is required when following the link, this means that any email address at such a domain is automatically confirmed without any human interaction, posing both a security flaw and a UI problem because the user will get a confusing error message when they follow that link manually.

We will need to work with the COmanage maintainers to either require authentication to confirm the email address or to require a button that one has to click rather than doing the confirmation automatically.

User authentication
-------------------

We will point Gafaelfawr_ for a Rubin Science Platform instance directly at CILogon and not configure CILogon to know about the contents of COmanage.
It will therefore be the responsibility of Gafaelfawr, when processing a user login via CILogon, to confirm via the user information API that the user has a valid account and to send them through the onboarding flow if they don't.
Gafaelfawr will have the CILogon unique identifier, so the user information API will need to support queries based on that and return the appropriate username or an error if the user is not registered.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

Open questions
==============

#. Evaluate Grouper as an alternative to COmanage Registry groups for self-managed groups (and possibly for system-managed groups).

#. Determine how to manage and expose unique GIDs.

#. We want each user to also be the sole member of a group by the same name.
   We can either create that group as a real group or artificially create it on the Science Platform side.
   (The latter may be preferrable to avoid a plethora of uneditable groups.)
   This requires enforcing uniqueness between group names and user names in some way.
   Determine how to do that.

#. What are the allowed characters in usernames and group names?
   How can we restrict that character set?
