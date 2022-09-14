:tocdepth: 1

.. sectnum::

Abstract
========

COmanage_ is a user management system designed for academic and research collaborations.
Rubin Observatory will use COmanage (as provided by CILogon_) as the user identity store and group management system for the Rubin Science Platform.
This tech note provides the specific details of the COmanage configuration used by the Science Platform and summarizes remaining COmanage work.

.. _COmanage: https://www.incommon.org/software/comanage/
.. _CILogon: https://cilogon.org/

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are DMTN-234_, which describes the high-level design; DMTN-224_, which describes the implementation; and SQR-069_, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _DMTN-234: https://dmtn-234.lsst.io/
.. _DMTN-224: https://dmtn-224.lsst.io/
.. _SQR-069: https://sqr-069.lsst.io/

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

Configure enrollment flow
-------------------------

Note that we use the preferred email and full name, not the official one.
This must match the settings used during :ref:`LDAP provisioning <ldap-provisioning>`.

.. rst-class:: compact

#. Edit "Self Signup With Approval" enrollment flow
#. Change "Email Confirmation Mode" to ``review`` and save
#. Edit its enrollment attributes
#. Edit the Name attribute, change its attribute definition to Preferred rather than Official, and make sure that only Given Name is required
#. Edit the Email attribute and change its attribute definition to Preferred rather than Official
#. Add username with a suitable description.
   Allow the user to change it during enrollment.
   Set the type of the field to CO Person, Identifier, UID.
   Mark as required.

The email confirmation mode setting adds a confirmation screen when confirming an email address.
If this is not done, just visiting the URL sent in an email address will automatically confirm the email address.
This interacts poorly with email anti-virus systems that retrieve all URLs in incoming messages and thus would automatically confirm email addresses.
Since anti-virus systems don't interact with the retrieved page, requiring the user click a button addresses this problem.

The above approach for selecting usernames does not work for the "Invite a collaborator" enrollment flow, since the person creating the invite is prompted for the username (this is CO-1002_).
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

Unlike with usernames, COmanage does not provide out-of-the-box support for validating group names with a regular expression.
We therefore use a custom plugin to enforce the group naming constraints defined in DMTN-225_.

The plugin used is `GroupNameValidator <https://github.com/cilogon/GroupNameValidator>`__ with the following configuration:

.. code-block:: php

   Configure::write('GroupNameValidator.pattern', '/^g_[a-z0-9._-]{1,30}$/');
   Configure::write('GroupNameValidator.flash_error_text', 'Name must start with g_ and use only a-z,0-9,.,_, and -');

API
===

LDAP
----

To make LDAP queries, use commands like:

.. code-block:: console

   $ ldapsearch -LLL -H ldaps://ldap-test.cilogon.org \
                -D 'uid=readonly_user,ou=system,o=LSST,o=CO,dc=lsst,dc=org' \
                -x -w PASSWORD -b 'ou=people,o=LSST,o=CO,dc=lsst,dc=org'

For IDF dev, the DNs end in ``dc=lsst_dev,dc=org``.
For IDF int, the DNs end in ``dc=lsst,dc=org`` as shown above.

The password is in 1Password under the hostname of the COmanage registry.
Use ``ou=people,o=LSST,o=CO,dc=lsst,dc=org`` for people and ``ou=groups,o=LSST,o=CO,dc=lsst,dc=org`` for groups.

COmanage REST API
-----------------

Only the `REST v1 API <https://spaces.at.internet2.edu/display/COmanage/REST+API+v1>`__ is currently available.
The base URL is the hostname of the COmanage registry service with ``/registry`` appended.

We currently don't expect to use the REST API.

Gafaelfawr
==========

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
     userDn: "uid=readonly_user,ou=system,o=LSST,o=CO,dc=lsst_dev,dc=org"
     groupBaseDn: "ou=groups,o=LSST,o=CO,dc=lsst_dev,dc=org"
     groupObjectClass: "eduMember"
     groupMemberAttr: "hasMember"
     userBaseDn: "ou=people,o=LSST,o=CO,dc=lsst_dev,dc=org"
     userSearchAttr: "voPersonApplicationUID"
     addUserGroup: true

This uses the CILogon test LDAP server (a production configuration will probably use a different LDAP server) and links to an enrollment flow in a test version of COmanage.

Open COmanage work
==================

- If the user selects a username during enrollment that has already been taken by another user, they are left stranded at an error screen with no obvious way to proceed.
  They can use the back button in the browser to go back to the enrollment form and choose a different username, but it's not obvious that this is possible and inline validation would be better.
  The proposed solution is to gather the username using an enrollment flow plugin with its own screen and validation.
  This plugin already exists and just needs to be deployed and configured.

- COmanage comes with a bunch of default components that we don't want to use (announcement feeds, forums, etc.).
  Edit the default dashboard to remove those widgets and replace them with widges for group management and personal identity management (if there are any applicable ones).

- Not directly COmanage, but we would like to customize the CILogon UI.
  This is done by writing CSS intended to be layered on top of the `base CSS <https://cilogon.org/include/cilogon.css>`__ and providing it to the CILogon team so that they can store it in the database.
  It is then selected via a parameter to the login URL, which Gafaelfawr already supports.
  The current LSST CSS is attached to `DM-35698 <https://jira.lsstcorp.org/browse/DM-35698>`__ in Jira.
