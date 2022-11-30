:tocdepth: 1

Abstract
========

COmanage_ is a user management system designed for academic and research collaborations.
Rubin Observatory will use COmanage (as provided by CILogon_) as the user identity store and group management system for the Rubin Science Platform.
This tech note provides the specific details of the COmanage configuration used by the Science Platform and summarizes remaining COmanage work.

.. _COmanage: https://www.incommon.org/software/comanage/
.. _CILogon: https://cilogon.org/

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are :dmtn:`234`, which describes the high-level design; :dmtn:`224`, which describes the implementation; and :sqr:`069`, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

Administrators
==============

We use separate installations of COmanage for the three :abbr:`IDF (Interim Data Facility)` environments so that we can test configuration changes and integrations without breaking other environments.

Each installation has at least two collaborations: the COmanage internal collaboration, and the LSST collaboration.
The latter is where all the users are configured.
It may have additional collaborations in the future to support delegated group management for specific science collaborations or other subgroups of users.

There are, correspondingly, two types of administrators for the COmanage instance.
Platform Administrators are users who are members of the COmanage internal collaboration and the ``CO:admins`` group of that collaboration.
They can make any changes to any collaboration.

Regular administrators are members of the ``CO:admins`` group of the LSST collaboration, but are not members of the COmanage collaboration (and thus will not see it in the UI).
They can approve petitions (COmanage's term for approving new users), change user attributes, and manage any groups in the LSST collaboration, but don't have special powers outside of that collaboration.

It's possible to grant access to approve petitions to a different group by changing the enrollment flow configuration.
That plus making the people handling approval owners of any groups they may need to update would allow us to remove ``CO:admins`` permissions in favor of more limited access.
Currently, we haven't bothered with this.

.. warning::

   **Always use incognito or private browsing when juggling multiple identities in COmanage.**

   Administering COmanage often requires using multiple identities.
   For example, one may be a Platform Administrator under one identity and be approving and setting group membership for a separate identity for yourself that will be a regular administrator.

   It is very easy to get your COmanage record into an awkward or broken state if those identities aren't kept separate.
   Therefore, whenever manipulating Platform Administrators or onboarding yourself under a second identity as a distinct person (as opposed to adding a new identity to an existing record), always use separate incognito windows for each identity.

Platform administrators
-----------------------

COmanage strongly recommends using a separate identity for a Platform Administrator that is not a member of any other collaboration (and indeed, in the version of COmanage as of 2022-11-29, if a Platform Administrator is onboarded into the LSST collaboration, they lose their Platform Administrator powers).
We therefore use the Google ``*-admin`` accounts in the lsst.cloud Google Cloud Identity domain as the identities for Platform Administrators.
(See :rtn:`020` for more details about that domain and those accounts.)

Only a few people need to be Platform Administrators.
That access only needs to be used to make some configuration changes and to fix problems if other administrators are locked out.

Adding new Platform Administrators needs to be done manually, since there is no regular enrollment flow defined for the special COmanage collaboration.
Follow the `Default Registry Enrollment <https://spaces.at.internet2.edu/display/COmanage/Default+Registry+Enrollment>`__ instructions.
To get the login identifier used in step one, have the person being onboarded go to `CILogon <https://cilogon.org/>`__ in a fresh icognito window and log on with their ``*-admin`` account.
On the resulting screen, expand user attributes, and use the "CILogon User Identifier" string (which will look like a URL).
When adding this as an identifier to the Organizational Identity, use the "OIDC sub" identity type.

Once the user's person record has been created using that process, go to the COmanage collaboration, choose :guilabel:`Groups` from the sidebar, choose :guilabel:`All Groups`, and add them as a member and owner of the ``CO:admins`` group.

Regular administrators
----------------------

Regular administrators can be onboarded in the normal way, using the "Self Signup With Approval" flow like any other user.
Once their petition has been approved and their user record has been created, go to the LSST collaboration, choose :guilabel:`Groups` from the sidebar, choose :guilabel:`All Groups`, and add them as a member and owner of the ``CO:admins`` group.

This is the appropriate permissions for users who will be approving the petitions of other users and sorting users into the appropriate groups.

Configuration
=============

CILogon
-------

(Not strictly part of COmanage, but closely related.)

We use custom CSS for CILogon, which we then specify by passing ``skin=LSST`` in the parameters to the CILogon login page.
This has to be manually loaded by the CILogon folks.

The CSS we use is maintained in the `lsst-sqre/cilogon-theme GitHub project <https://github.com/lsst-sqre/cilogon-theme>`__.
``src/rubin.css`` is the file that we provide to CILogon.

This setup only has to be done once for all environments, not per-environment like the other COmanage configuration, and only needs to be redone if the CSS file changes.

See `DM-35698 <https://jira.lsstcorp.org/browse/DM-35698>`__ for the process we followed when updating the CSS.

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
#. Add a new enrollment attribute:
   - Label: ``Users group``
   - Attribute class: ``CO Person``
   - Attribute name: ``Group member``
   - Required: ``Required``
   - Default value: ``g_users`` (or whatever the name of the general users group is)
   - Modifiable: unchecked
   - Hidden: checked

The email confirmation mode setting adds a confirmation screen when confirming an email address.
If this is not done, just visiting the URL sent in an email address will automatically confirm the email address.
This interacts poorly with email anti-virus systems that retrieve all URLs in incoming messages and thus would automatically confirm email addresses.
Since anti-virus systems don't interact with the retrieved page, requiring the user click a button addresses this problem.

The additional enrollment attribute automatically adds new users to the general users group, avoiding an additional step for the person approving new users unless that user needs to be a member of a special group.

In addition, we install the `IdentifierEnroller Plugin <https://spaces.at.internet2.edu/display/COmanage/IdentifierEnroller+Plugin>`__ and use it to capture the requested username after email verification.
This plugin has better error handling than adding username to the list of enrollment attributes, particularly if that username is already in use.
It is attached as an enrollment flow wedge to the "Self Signup With Approval" enrollment flow.

To configure this plugin:

.. rst-class:: compact

#. Attach the IdentifierEnroller as a wedge
#. Select Configure
#. Select "Manage Indentifier Enroller Identifiers"
#. Set the label to ``Username``
#. Set the description to something appropriate
#. Set the identifier type to ``UID``

Username validation
-------------------

Ensure the `Regex Identifier Validator Plugin`_ is enabled.  Then:

.. rst-class:: compact

#. Go to Configuration → Identifier Validators and add a new validator
#. Set the name to "Username validation", the plugin to RegexIdentifierValidator, and the attribute to UID, and click Add
#. Set the regular expression to::

       /^[a-z0-9](?:[a-z0-9]|-[a-z0-9])*[a-z](?:[a-z0-9]|-[a-z0-9])*$/

This implements the restrictions on valid usernames documented in :dmtn:`225`.

.. _Regex Identifier Validator Plugin: https://spaces.at.internet2.edu/display/COmanage/Regex+Identifier+Validator+Plugin

.. _group-name-validation:

Group name validation
---------------------

Unlike with usernames, COmanage does not provide out-of-the-box support for validating group names with a regular expression.
We therefore use a custom plugin to enforce the group naming constraints defined in :dmtn:`225`.

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

- The landing pages before and after verifying the user's email address need further customization.
  The current versions are in the `cilogin/lsst-registry-landing GitHub repository <https://github.com/cilogon/lsst-registry-landing>`__.

- COmanage comes with a bunch of default components that we don't want to use (announcement feeds, forums, etc.).
  Edit the default dashboard to remove those widgets and replace them with widges for group management and personal identity management (if there are any applicable ones).
