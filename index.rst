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

Configure LDAP provisioning target
----------------------------------

#. Go to Configuration -> Provisioning Targets and configure Primary LDAP
#. Go down to the attribute configuration
#. Enable ``displayName``, disable ``givenName``, and set it to Official
#. Change ``uid`` to use the UID identifier
#. Enable ``groupOfNames`` objectclass
#. Enable ``hasMember`` in the ``eduMember`` objectclass
#. Enable ``voPerson`` and set ``voPersonID`` to the LSST Registry ID
#. Save and then Reprovision All to update existing records

Dashboard
---------

COmanage comes with a bunch of default components that we probably don't want to use (announcement feeds, forums, etc.).
We will want to edit the default dashboard to remove those widges and replace them with widges for group management and personal identity management (if there are any applicable ones).

Integration
===========

API interfaces
--------------

Only the `REST v1 API <https://spaces.at.internet2.edu/display/COmanage/REST+API+v1>`__ is currently available.
The base URL is the hostname of the COmanage registry service with ``/registry`` appended.

To make LDAP queries, use commands like:

.. code-block:: console

   $ ldapsearch -LLL -H ldaps://ldap-test.cilogon.org \
                -D 'uid=readonly_user,ou=system,o=LSST,o=CO,dc=lsst,dc=org' \
                -x -w PASSWORD -b 'ou=people,o=LSST,o=CO,dc=lsst,dc=org'

The password is in 1Password under the hostname of the COmanage registry.

Example LDAP query results
--------------------------

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

An example group::

    dn: cn=science-platform-idf-dev,ou=groups,o=LSST,o=CO,dc=lsst,dc=org
    cn: science-platform-idf-dev
    member: voPersonID=LSST100006,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    objectClass: groupOfNames
    objectClass: eduMember
    hasMember: rra

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

User onboarding API
-------------------

When a new user first accesses the Rubin Science Platform, we will need to route them through the onboarding flow, and then may need to make additional changes to their record via the COmanage API.
This can be integrated with the onboarding service described in SQR-052_.
This service would have a privileged API token for the Rubin Science Platform COmanage environment.

.. _SQR-052: https://sqr-052.lsst.io/

Open questions
==============

#. How should we allocate numeric GIDs and expose them in LDAP?

#. If we enable the ``voPosixGroup`` schema, where does the ``voPosixAccountGidNumber`` attribute come from?

#. How can we restrict inbound CILogon authentications to the Rubin Science Platform to only registered COmanage users?
   Is this functionality that's built into CILogon and COmanage in some way, or will the Rubin Science Platform authentication layer need to check COmanage to see if the user already exists and send them into an account creation flow if they do not?
