:tocdepth: 1

.. sectnum::

Abstract
========

Rubin Observatory plans to use COmanage as the identity management platform for the Rubin Science Platform.
Group membership will be maintained inside COmanage.
UID and GID assignment will be done outside of COmanage by Gafaelfawr_, the authentication and authorization component of the Rubin Science Platform.
This tech note provides a step-by-step how to configure COmanage for that purpose, discusses alternate configuration approaches that were discarded, discusses integration requirements, and summarizes remaining COmanage work.

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are DMTN-234_, which describes the high-level design; DMTN-224_, which describes the implementation; and SQR-069_, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _DMTN-234: https://dmtn-234.lsst.io/
.. _DMTN-224: https://dmtn-224.lsst.io/
.. _SQR-069: https://sqr-069.lsst.io/

Decisions
=========

#. **Use COmanage for group management.**
   Grouper offers more features, but at the cost of additional user complexity, a UI that's not better than the COmanage UI, a less usable API, and additional integration and conceptual complexity.
   We will need to add a plugin to do group name validation (see :ref:`Group name validation <group-name-validation>`).

#. **Assign UIDs and GIDs outside of COmanage.**
   It's possible to manage both inside COmanage, using ``voPosixGroup`` to do GID assignment and an auto-incrementing unique ID to do UID assignment (as a string that would require some postprocessing).
   However, this doesn't let us enforce range boundaries or use a different range for bot users.
   We will instead use Google Firestore to store UIDs and GIDs.

#. **Use LDAP as the primary API to retrieve user and group data.**
   Despite requiring an LDAP library dependency, this looks easier to use than the COmanage or Grouper APIs.

#. **Implement quota rules outside of COmanage.**
   We have to maintain a local database anyway for the token system and can put quota metadata in the same place.
   Neither COmanage nor Grouper easily support quota math.
   The Grouper API could allow us to use it as a backing store for quota data, but the API is sufficiently hard to use that this isn't an attractive option.

#. **Manage user metadata in Gafaelfawr.**
   Gafaelfawr_ will gather data from LDAP on demand, with a local cache to reduce the LDAP load, and merge that with data stored with the token.
   This can be extended to support quotas when those are added.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

Configuration
=============

Configure unique attribute for each person
------------------------------------------

CILogon generates a unique identifier for every authentication identity.
COmanage may map multiple authentication identities to the same person record (so that someone can log in via both GitHub and their local university, for example, which are separate authentication identities).
To resolve a login identity to a person in LDAP, those authentication identities must be present in the LDAP record.
The recommended attribute in which to store them is ``uid``, which is multivalued.

This means that a different attribute must be used for the unique identifier for each person.
That attribute must not be multivalued.
We use ``LSST Registry ID`` as that attribute.

.. rst-class:: compact

#. Go to Configuration → Extended Types
#. Add an extended type:

   - Name: ``lsstregistryid``
   - Display Name: ``LSST Registry ID``

#. Go to Configuration → Identifier Assignments
#. Create an identifier assignment:

   - Description: ``LSST Registry ID``
   - Context: ``CO Person``
   - Type: ``LSST Registry ID`` (do not check the Login box)
   - Algorithm: ``Sequential``
   - Format: ``LSST(#)`` (via "Select a common pattern")
   - Permitted Characters: ``AlphaNumeric Only``
   - Minimum: ``1000000`` (this doesn't really matter but it will make all the identifiers the same length)

.. _ldap-provisioning:

Configure LDAP provisioning target
----------------------------------

Gafaelfawr requires the bare usernames be listed in an attribute in each group record so that they can be searched for.
This is supported by the ``eduMember`` object class and the ``hasMember`` attribute.

We also need to put the user's canonical username (which COmanage calls the UID) in some attribute in the person record so that we can query on it.
We use ``voPersonApplicationUID`` for that purpose.
Note that this is different than ``uid`` (the CILogon federated identity strings) and the unique attribute for each person used in the person data hierarchy (``voPersonID``).

Note that we use the preferred email and full name, not the official one.
This must match the settings used during :ref:`enrollment flow <enrollment-flow>`.

.. rst-class:: compact

#. Go to Configuration → Provisioning Targets and configure Primary LDAP
#. Set "People DN Identifier Type" to ``LSST Registry ID``
#. Set "People DN Attribute Name" to ``voPersonId``
#. Go down to the attribute configuration
#. Enable ``displayName``, disable ``givenName``, and set it to Preferred
#. Change ``mail`` to Preferred
#. Change ``uid`` to OIDC sub and select the box for "Use value from Organizational Identity"
#. Enable ``groupOfNames`` objectclass
#. Enable ``isMemberOf`` in the ``eduMember`` objectclass
#. Enable ``hasMember`` in the ``eduMember`` objectclass and set it to UID
#. Enable ``voPerson`` objectclass

   #. Enable ``voPersonApplicationUID`` and set it to UID
   #. Enable ``voPersonID`` and set it to LSST Registry ID
   #. Enable ``voPersonSoRID`` and set it to System of Record ID

#. Save and then Reprovision All to update existing records

OpenID Connect client
---------------------

The important configuration setting here is to map ``voPersonApplicationUID`` to the ``username`` claim.
This is used by Gafaelfawr_ to get the username after authentication or, if that claim is not set, to know that the user is not enrolled in COmanage and to redirect to an enrollment flow.

.. rst-class:: compact

#. Go to Configuration → OIDC Clients
#. Add a new client
#. Set the name to a reasonable short description of the deployment
#. Set the home URL to the top-level URL of the deployment
#. Set the callback to the home URL with ``/login`` appended (the Gafaelfawr callback URL)
#. Enable the ``org.cilogon.userinfo`` scope
#. Add an LDAP to claim mapping

   - LDAP attribute name: ``voPersonApplicationUID``
   - OIDC Claim Name: ``username``

.. _enrollment-flow:

Add username to enrollment flow
-------------------------------

Note that we use the preferred email and full name, not the official one.
This must match the settings used during :ref:`LDAP provisioning <ldap-provisioning>`.

.. rst-class:: compact

#. Edit "Self Signup With Approval" enrollment flow
#. Edit its enrollment attributes
#. Edit the Name attribute, change its attribute definition to Preferred rather than Official, and make sure that only Given Name is required
#. Edit the Email attribute and change its attribute definition to Preferred rather than Official
#. Add username with a suitable description.
   Allow the user to change it during enrollment.
   Set the type of the field to CO Person, Identifier, UID.
   Mark as required.

This does not work for the "Invite a collaborator" enrollment flow, since the person creating the invite is prompted for the username (this is CO-1002_).
We probably won't need that flow.
If we do, we'll need a separate enrollment flow plugin (which does not exist as a turnkey configuration, but there are examples to work from) to collect the username after email validation.

.. _CO-1002: https://todos.internet2.edu/browse/CO-1002

Username validation
-------------------

Ensure the `Regex Identifier Validator Plugin`_ is enabled.  Then:

.. rst-class:: compact

#. Go to Configuration → Identifier Validators and add a new validator
#. Set the name to "Username validation", the plugin to RegexIdentifierValidator, and the attribute to UID, and click Add
#. Set the regular expression to::

       /^[a-z0-9](?:[a-z0-9]|-[a-z0-9])*[a-z](?:[a-z0-9]|-[a-z0-9])*$/

This implements the restrictions on valid usernames documented in `DMTN-225`_.

.. _Regex Identifier Validator Plugin: https://spaces.at.internet2.edu/display/COmanage/Regex+Identifier+Validator+Plugin
.. _DMTN-225: https://dmtn-225.lsst.io/

.. _group-name-validation:

Group name validation
---------------------

One approach is to use the `Group Name Filter Plugin`_.
Ensure it is also enabled.
Then:

.. rst-class:: compact

#. Go to Configuration → Extended Types and add a new type
#. Set the name to "groupname" and the display name to "Group name"
#. Go to Configuration → Data Filters and add a new filter
#. Set the name to "Force group name validation" and the plugin to GroupNameFilter and click Add
#. Set the identifier type to "Group name"
#. Go to Configuration → Identifier Validators and add a new validator
#. Set the name to "Username validation", the plugin to RegexIdentifierValidator, and the attribute to UID, and click Add
#. Set the regular expression to::

       /^g_[a-z0-9._-]{1,30}$/

This essentially replaces the group name with an identifier and requires that identifier to start with ``g_``, which will avoid conflicts between usernames and groups.
`DMTN-225`_ defines the constraints on group names.

.. _Group Name Filter Plugin: https://spaces.at.internet2.edu/display/COmanage/Group+Name+Filter+Plugin

However, this doesn't change the group creation flow.
One has to explicitly go into the group and add the new Group name identifier.

A better approach would be a CakePHP plugin that intercepts the save call and can enforce a group naming convention.
This would use the `CakePHP Event System`_.
This plugin does not already exist, but the CILogon folks have a previously-written plugin that is very similar and could adapt it to our needs.

.. _CakePHP Event System: https://book.cakephp.org/2/en/core-libraries/events.html

Dashboard
---------

COmanage comes with a bunch of default components that we don't want to use (announcement feeds, forums, etc.).
We will want to edit the default dashboard to remove those widges and replace them with widges for group management and personal identity management (if there are any applicable ones).

Other configurations considered
===============================

Group management
----------------

We have two primary options for managing groups via COmanage: using COmanage Registry groups, or using Grouper.
In both cases, there are limitations on how much we can customize the UI without a lot of development.

Quota calculation is not directly supported with either system and in either case would need custom development (either via a plugin or via a service that used the group API).
Recording quota information for groups locally and using the group API (or LDAP) to synchronize the list of groups with the canonical list looks like the easiest path.

COmanage Registry groups
^^^^^^^^^^^^^^^^^^^^^^^^

Advantages:

.. rst-class:: compact

#. Uses the same UI as the onboarding and identity management process
#. Possible (albeit complex) to automatically generate GIDs using ``voPosixGroup`` (see :ref:`voPosixGroup <voposixgroup>`)

Disadvantages:

.. rst-class:: compact

#. No support for nested groups
#. Groups cannot own other groups
#. No support for set math between groups
#. No generic metadata support, so group quotas would need to be maintained separately (presumably by a Rubin-developed service)
#. There currently is a rendering bug that causes each person to show up three times when editing the group membership, but this will be fixed in the 4.0.0 release due in the second quarter of 2021

Grouper
^^^^^^^

Advantages:

.. rst-class:: compact

#. Full support for nested groups
#. Groups can own other groups
#. Specializes in set math between groups if we want to do complex authorization calculations
#. Arbitrary metadata can be added to groups via the API, so we could use Grouper as our data store rather than a local database

Disadvantages:

.. rst-class:: compact

#. More complex setup and data flow
#. Users have to interact with two UIs, the COmanage one for identities and the Grouper UI for group management
#. No support for automatic GID generation

.. _gid:

Numeric GIDs
------------

Getting numeric GIDs into the LDAP entries for each group isn't well-supported by COmanage.
The LDAP connector does not have an option to add arbitrary group identifiers to the group LDAP entry.

We decided to avoid this problem by assigning UIDs and GIDs outside of COmanage.
Here are a few other possible options we considered.

COmanage group REST API
^^^^^^^^^^^^^^^^^^^^^^^

Arbitrary identifiers can be added to groups, so a group can be configured with an auto-incrementing unique identifier in the same way that we do for users, using a base number of 200000 instead of 100000 to keep the UIDs and GIDs distinct (allowing the UID to be used as the GID of the primary group).
Although that identifier isn't exposed in LDAP, it can be read via the COmanage REST API using a URL such as::

    https://<registry-url>/registry/identifiers.json?cogroupid=7

The group ID can be obtained from the ``/registry/co_groups.json`` route, searching on a specific ``coid``.
Middleware running on the Rubin Science Platform could cache the GID information for every group, refresh it periodically, and query for the GID of a new group when seen.

.. _voposixgroup:

voPosixGroup
^^^^^^^^^^^^

Another option is to enable ``voPosixGroup`` and generate group IDs that way.
However, that process is somewhat complex.

COmanage Registry has the generic notion of a `Cluster <https://spaces.at.internet2.edu/display/COmanage/Clusters>`__.
A Cluster is used to represent a CO Person's accounts with a given application or service.

Cluster functionality is implemented by Cluster Plugins.
Right now there is one Cluster Plugin that comes out of the box with COmanage, the `UnixCluster plugin <https://spaces.at.internet2.edu/display/COmanage/Unix+Cluster+Plugin>`__.

The UnixCluster plugin is configured with a "GID Type."
From the documentation: "When a CO Group is mapped to a Unix Cluster Group, the CO Group Identifier of this type will be used as the group's numeric ID."
CO Person can then have a UnixCluster account that has associated with it a UnixCluster Group, and the group will have a GID identifier.

To have the information about the UnixCluster and the UnixCluster Group provisioned into LDAP using the ``voPosixAccount`` objectClass, define a `CO Service <https://spaces.at.internet2.edu/display/COmanage/Registry+Services>`__ for the UnixCluster.
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
With a second UnixCluster and CO Service with short label "slac" to represent an account at SLAC, this record would have additionally::

    voPosixAccountGidNumber;scope-slac: 1000001

The UnixCluster object and UnixCluster Group objects and all the identifiers are usually established during an enrollment flow.

Grouper
^^^^^^^

Grouper does not have built-in support for assigning numeric GIDs to each group out of some range.
It is possible to cobble something together using the ``idIndex`` that Grouper generates (see `this discussion <https://lists.internet2.edu/sympa/arc/grouper-users/2017-01/msg00087.html>`__ and `this documentation <https://spaces.at.internet2.edu/display/Grouper/Integer+IDs+on+Grouper+objects>`__), but it would require some development.

Alternately, groups can be assigned arbitrary attributes that we define, so we can assign GIDs to groups via the API, but we would need to maintain the list of available GIDs and ensure there are no conflicts.
Grouper also does not appear to care if the same attribute value is assigned to multiple groups, so we would need to handle uniqueness.

Custom development
^^^^^^^^^^^^^^^^^^

We could enhance (or pay someone to enhance) the LDAP Provisioning Plugin to allow us to express an additional object class in the group tree in LDAP, containing a numeric GID identifier.

API
===

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
    displayName: Russ Allbery
    sn: Allbery
    cn: Russ Allbery
    objectClass: person
    objectClass: organizationalPerson
    objectClass: inetOrgPerson
    objectClass: eduMember
    objectClass: voPerson
    uid: http://cilogon.org/serverA/users/31388556
    uid: http://cilogon.org/serverA/users/15423111
    isMemberOf: CO:members:all
    isMemberOf: CO:members:active
    isMemberOf: CO:admins
    isMemberOf: g_science-platform-idf-dev
    isMemberOf: g_test-group
    voPersonApplicationUID: rra
    voPersonID: LSST100006
    voPersonSoRID: http://cilogon.org/serverA/users/31388556

An example group::

    dn: cn=g_science-platform-idf-dev,ou=groups,o=LSST,o=CO,dc=lsst,dc=org
    cn: g_science-platform-idf-dev
    member: voPersonID=LSST100006,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    member: voPersonID=LSST100007,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    member: voPersonID=LSST100008,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    member: voPersonID=LSST100010,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    member: voPersonID=LSST100012,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    member: voPersonID=LSST100013,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    objectClass: groupOfNames
    objectClass: eduMember
    hasMember: rra
    hasMember: thoron
    hasMember: frossie
    hasMember: cbanek
    hasMember: afausti
    hasMember: simonkrughoff

COmanage REST API
-----------------

Only the `REST v1 API <https://spaces.at.internet2.edu/display/COmanage/REST+API+v1>`__ is currently available.
The base URL is the hostname of the COmanage registry service with ``/registry`` appended.

We currently don't expect to use the REST API.

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

We didn't investigate this further since we decided against using Grouper for group management.

Integration
===========

On the Rubin Science Platform side, we will need to implement the following.

User information
----------------

Gafaelfawr_ will be set up to use OpenID Connect for authentication, using the OIDC client information configured above.
It will take the authenticated username from the ``username`` claim of the token, and then look up other information about the user (group membership, full name, email address) from LDAP on demand with a short-lived cache.
(UIDs and GIDs will be handled externally from COmanage in Firestore.)

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
This has not yet been done.

It's not clear yet whether we will need to automate additional changes to a person's record after onboarding, such as adding them to groups, or if this will be handled manually during the approval process.
If we do need to automate this, we may need to do that via the COmanage API.

The web pages shown during this onboarding flow are controlled by the style information in the `lsst-registry-landing <https://github.com/cilogon/lsst-registry-landing>`__ project on GitHub.

Email verification issue
^^^^^^^^^^^^^^^^^^^^^^^^

Currently, user onboarding has a bug: After choosing their name, email, and username, the user is sent an email message to confirm that they have control over that email address.
The link in the mail message has a one-time code in it, and confirms the email address when followed.
However, sites with anti-virus integrated with their email system (such as AURA) often pre-fetch all URLs seen in email addresses.
Since no authentication or confirmation is required when following the link, this means that any email address at such a domain is automatically confirmed without any human interaction, posing both a security flaw and a UI problem because the user will get a confusing error message when they follow that link manually.

We will need to work with the COmanage maintainers to either require authentication to confirm the email address or to require a button that one has to click rather than doing the confirmation automatically.

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

.. _DMTN-182: https://dmtn-182.lsst.io/

These groups will not be managed in COmanage or Grouper.
They will be synthesized by Gafaelfawr_ in response to queries about the user.
(This work is not yet done.)

Example Gafaelfawr configuration
--------------------------------

Here is an example configuration of the Gafaelfawr Helm chart to use CILogon and COmanage.
This is suitable for the ``values-*.yaml`` file in Phalanx_.

.. _Phalanx: https://phalanx.lsst.io/

.. code-block:: yaml

   cilogon:
     clientId: "cilogon:/client_id/46f9ae932fd30e9fb1b246972a3c0720"
     enrollmentUrl: "https://registry-test.lsst.codes/registry/co_petitions/start/coef:6"
     usernameClaim: "username"

   firestore:
     project: "rsp-firestore-dev-31c4"

   ldap:
     url: "ldaps://ldap-test.cilogon.org"
     userDn: "uid=readonly_user,ou=system,o=LSST,o=CO,dc=lsst,dc=org"
     groupBaseDn: "ou=groups,o=LSST,o=CO,dc=lsst,dc=org"
     groupObjectClass: "eduMember"
     groupMemberAttr: "hasMember"
     userBaseDn: "ou=people,o=LSST,o=CO,dc=lsst,dc=org"
     userSearchAttr: "voPersonApplicationUID"

This uses the CILogon test LDAP server (a production configuration will probably use a different LDAP server) and links to an enrollment flow in a test version of COmanage.

Open COmanage work
==================

#. Add a button or require authentication before confirming the email address to avoid a bug in the onboarding flow.

#. Write a CakePHP plugin to enforce a group naming convention.
