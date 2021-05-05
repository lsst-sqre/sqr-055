:tocdepth: 1

.. sectnum::

Abstract
========

We plan to use COmanage as the identity management platform for the Rubin Science Platform.
This document describes how to configure COmanage for that purpose and proposes a design for the necessary integration services to retrieve user metadata, group membership, and group metadata.

Recommendations
===============

#. **Use COmanage for group management.**
   Grouper offers more features, but at the cost of additional user complexity, a UI that's not better than the COmanage UI, a less usable API, and additional integration and conceptual complexity.
   However, COmanage does not appear to provide usable name validation (see :ref:`group-name-validation`), which is an argument for either writing a plugin that would do so or managing groups ourselves rather than using either Grouper or COmanage.

#. **Use ``voPosixGroup`` for numeric GIDs.**
   This assumes that we want to use COmanage to manage groups.
   The complexity level is unfortunate, but it appears to mostly be one-time configuration complexity, and there are substantial implementation advantages to supporting automatic assignment of new GIDs without having to write custom code.
   The alternative would be to use a custom identifier, but since it can't be expressed in LDAP, this would require using the REST API to retrieve group data, which adds ongoing rather than one-time complexity.

#. **Use LDAP as the primary API to retrieve user and group data.**
   Despite requiring an LDAP library dependency, this looks easier to use than the COmanage or Grouper APIs.
   It will also be the primary source of GIDs given the use of ``voPosixGroup``.

#. **Implement quota rules outside of COmanage.**
   We have to maintain a local database anyway for the token system and can put quota metadata in the same place.
   Neither COmanage nor Grouper easily support quota math.
   The Grouper API could allow us to use it as a backing store for quota data, but the API is sufficiently hard to use that this isn't an attractive option.

#. **Write a local service to return user metadata.**
   That API service gather data from LDAP and provide the equivalent of the current Gafaelfawr ``/auth/api/v1/user-info`` endpoint, which returns full name, email address, numeric UID, and group information (names and GIDs).
   That service can be extended to support quotas when those are added.
   Gafaelfawr will then query that service as part of the ``/auth`` endpoint (probably with local caching for performance).

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

This does not work for the "Invite a collaborator" enrollment flow, since the person creating the invite is prompted for the username (this is `CO-1002`_).
We probably won't need that flow.
If we do, we'll need a separate enrollment flow plugin (which does not exist as a turnkey configuration, but there are examples to work from) to collect the username after email validation.

.. _CO-1002: https://todos.internet2.edu/browse/CO-1002

Configure LDAP provisioning target
----------------------------------

#. Go to Configuration → Provisioning Targets and configure Primary LDAP
#. Go down to the attribute configuration
#. Enable ``displayName``, disable ``givenName``, and set it to Official
#. Change ``uid`` to use the UID identifier
#. Enable ``groupOfNames`` objectclass
#. Enable ``hasMember`` in the ``eduMember`` objectclass
#. Enable ``voPerson`` and set ``voPersonID`` to the LSST Registry ID and ``voPersonSoRID`` to System of Record ID
#. Save and then Reprovision All to update existing records

Username validation
-------------------

Ensure the `Regex Identifier Validator Plugin`_ is enabled.  Then:

#. Go to Configuration → Identifier Validators and add a new validator
#. Set the name to "Username validation", the plugin to RegexIdentifierValidator, and the attribute to UID, and click Add
#. Set the regular expression to::

       /^[a-z0-9](?:[a-z0-9]|-[a-z0-9])*[a-z](?:[a-z0-9]|-[a-z0-9])*$/

We use the UID as a username, and this restricts the usernames to the valid values for a GitHub username while disallowing single-character usernames and usernames that are entirely numbers.

.. _Regex Identifier Validator Plugin: https://spaces.at.internet2.edu/display/COmanage/Regex+Identifier+Validator+Plugin

.. _group-name-validation:

Group name validation
---------------------

Ensure the `Group Name Filter Plugin`_ is also enabled.  Then:

#. Go to Configuration → Extended Types and add a new type
#. Set the name to "groupname" and the display name to "Group name"
#. Go to Configuration → Data Filters and add a new filter
#. Set the name to "Force group name validation" and the plugin to GroupNameFilter and click Add
#. Set the identifier type to "Group name"
#. Go to Configuration → Identifier Validators and add a new validator
#. Set the name to "Username validation", the plugin to RegexIdentifierValidator, and the attribute to UID, and click Add
#. Set the regular expression to::

       /^g_[a-z][a-z0-9_-]*$/

This essentially replaces the group name with an identifier and requires that identifier to start with ``g_``, which will avoid conflicts between usernames and groups.

.. _Group Name Filter Plugin: https://spaces.at.internet2.edu/display/COmanage/Group+Name+Filter+Plugin

Unfortunately, this doesn't seem to work.
No changes appear to the group name in LDAP.
It also doesn't change the group creation flow; one has to explicitly go into the group and add the new Group name identifier.

Dashboard
---------

COmanage comes with a bunch of default components that we don't want to use (announcement feeds, forums, etc.).
We will want to edit the default dashboard to remove those widges and replace them with widges for group management and personal identity management (if there are any applicable ones).

Group management
================

We have two primary options for managing groups via COmanage: using COmanage Registry groups, or using Grouper.
In both cases, there are limitations on how much we can customize the UI without a lot of development.

Quota calculation is not directly supported with either system and in either case would need custom development (either via a plugin or via a service that used the group API).
Recording quota information for groups locally and using the group API (or LDAP) to synchronize the list of groups with the canonical list looks like the easiest path.

COmanage Registry groups
------------------------

Advantages:

- Uses the same UI as the onboarding and identity management process
- Possible (albeit complex) to automatically generate GIDs using ``voPosixGroup`` (see :ref:`voposixgroup`)

Disadvantages:

- No support for nested groups
- Groups cannot own other groups
- No support for set math between groups
- No generic metadata support, so group quotas would need to be maintained separately (presumably by a Rubin-developed service)
- There currently is a rendering bug that causes each person to show up three times when editing the group membership, but this will be fixed in the 4.0.0 release due in the second quarter of 2021

Grouper
-------

Advantages:

- Full support for nested groups
- Groups can own other groups
- Specializes in set math between groups if we want to do complex authorization calculations
- Arbitrary metadata can be added to groups via the API, so we could use Grouper as our data store rather than a local database

Disadvantages:

- More complex setup and data flow
- Users have to interact with two UIs, the COmanage one for identities and the Grouper UI for group management
- No support for automatic GID generation

Numeric GIDs
============

Getting numeric GIDs into the LDAP entries for each group isn't well-supported by COmanage.
The LDAP connector does not have an option to add arbitrary group identifiers to the group LDAP entry.
There are a few possible options.

COmanage group REST API
-----------------------

Arbitrary identifiers can be added to groups, so a group can be configured with an auto-incrementing unique identifier in the same way that we do for users, using a base number of 200000 instead of 100000 to keep the UIDs and GIDs distinct (allowing the UID to be used as the GID of the primary group).
Although that identifier isn't exposed in LDAP, it can be read via the COmanage REST API using a URL such as::

    https://<registry-url>/registry/identifiers.json?cogroupid=7

The group ID can be obtained from the ``/registry/co_groups.json`` route, searching on a specific ``coid``.
Middleware running on the Rubin Science Platform could cache the GID information for every group, refresh it periodically, and query for the GID of a new group when seen.

.. _voposixgroup:

voPosixGroup
------------

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

Grouper
-------

Grouper does not have built-in support for assigning numeric GIDs to each group out of some range.
It is possible to cobble something together using the ``idIndex`` that Grouper generates (see `this discussion <https://lists.internet2.edu/sympa/arc/grouper-users/2017-01/msg00087.html>`__ and `this documentation <https://spaces.at.internet2.edu/display/Grouper/Integer+IDs+on+Grouper+objects>`__), but it would require some development.

Alternately, groups can be assigned arbitrary attributes that we define, so we can assign GIDs to groups via the API, but we would need to maintain the list of available GIDs and ensure there are no conflicts.
Grouper also does not appear to care if the same attribute value is assigned to multiple groups, so we would need to handle uniqueness.

Custom development
------------------

We could enhance (or pay someone to enhance) the LDAP Provisioning Plugin to allow us to express an additional object class in the group tree in LDAP, containing a numeric GID identifier.

API
===

COmanage REST API
-----------------

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

Grouper REST API
----------------

Grouper supports a REST API.
However, it appears to be very complex and documented primarily as a Java API.
I was unable to locate a traditional REST API description for it.
The API looks to be fully functional but it makes a number of unusual choices, such as ``T`` and ``F`` strings instead of proper booleans.

Using the API appears to require a lot of reverse engineering from example traces.
See, for instance, the `example of assigning an attribute value to a group <https://github.com/Internet2/grouper/blob/master/grouper-ws/grouper-ws/doc/samples/assignAttributesWithValue/WsSampleAssignAttributesWithValueRestLite_json.txt>`__.

A sample Grouper API call:

.. code-block:: console

   $ curl --silent -u GrouperSystem:XXXXXXXX \
     'https://group-registry-test.lsst.codes/grouper-ws/servicesRest/json/v2_5_000/groups/etc%3Asysadmingroup/members' \
     | jq .

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

Gafaelfawr will need to retrieve information about a user from this service to expose it in the headers that are passed as part of an authenticated request.
This is how we pass metadata to services that are not Gafaelfawr-aware and willing to make their own API calls, or to services that should not receive delegated tokens.

This corresponds primarily to the ``/auth/api/v1/user-info`` route specified in SQR-049_.

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

User-chosen usernames must meet the following requirements (the same as GitHub):

* Only alphanumerics and hyphen
* No two consecutive hyphens
* Username may not start or end with a hyphen
* Username may not be all digits

This can be enforced by COmanage.

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

User authorization
------------------

COmanage does not preserve the affiliation information sent by the identity provider, if any.
Affiliation in COmanage must be set to one of a restricted set of values, and the affiliation given by identity providers is free-form.
In our test instance, the affiliation was forced to always be "affiliate" to avoid this problem.

If we want to make use of the affiliation sent by the upstream identity provider for authorization decisions, we will have to write a COmanage plugin.
The difficult part of that is defining what the business logic should be.

To see the affiliation attributes sent by an identity provider, go directly to `CILogon <https://cilogon.org/>`__ and log on via that provider.
On the resulting screen, look at the User Attributes section.

User self groups
----------------

Each user will appear to the Rubin Science Platform to also be the sole member of a group with the same name as the username and the same GID as the UID.
This is a requirement for POSIX file systems underlying the Notebook Aspect and for the Butler service (see DMTN-182_ for the latter).

These groups will not be managed in COmanage or Grouper.
They will be synthesized by the group API maintained as part of the Science Platform.

.. _DMTN-182: https://dmtn-182.lsst.io/

Group naming
------------

Since each username must also correspond to a (synthesized) group name, we must avoid naming conflicts between users and groups.
We will do this by requiring all self-service group names start with ``g_``.
Since underscore (``_``) is not a valid character in usernames, this will avoid any conflicts.

Open COmanage work
==================

#. Add a button or require authentication before confirming the email address to avoid a bug in the onboarding flow.

#. The approach of replacing the group name with a different identifier in order to apply name validation doesn't appear to work.
   There doesn't seem to be a mechanism to prompt the user for that identifier when creating the group, and the identifier, when manually added, doesn't seem to change the name of the group provisioned to LDAP.

#. The 4.0.0 release of COmanage is supposed to resolve the duplicated user identities in the group management screen.
