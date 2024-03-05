#################################################
COmanage configuration for Rubin Science Platform
#################################################

.. abstract::

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

As mentioned above, make sure not to attempt to onboard this same identity into the LSST collaboration or add it as an additional entity to any person record in that collaboration, since this will currently cause it to lose Platform Administrator permissions.

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

One of the things this skin does is hide the "Remember me" checkbox from the login page.
This normally allows users to tell CILogon to remember which identity provider they use, so they never see the page to select an identity provider again.
Unfortunately, we've found this causes confusion in practice, since users end up wanting to select a different identity provider and can't, without going to an obscure-to-users CILogon page to remove that cookie.
We therefore disable that button with CSS so that the user always sees the identity provider selection page.
Their last selection is still remembered and selected by default.

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

#. Go to :menuselection:`Configuration --> Extended Types`
#. Add an extended type:

   - Name: ``lsstregistryid``
   - Display Name: ``LSST Registry ID``

#. Go to :menuselection:`Configuration --> Identifier Assignments`
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

#. Go to :menuselection:`Configuration --> Provisioning Targets` and configure Primary LDAP
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

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

.. rst-class:: compact

#. Go to :menuselection:`Configuration --> OIDC Clients`
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
#. Edit the Name attribute

   - Set :guilabel:`Description` to "Enter the name you are known by professionally (for example, the name you would use on scientific papers)"
   - Change its attribute definition to Preferred rather than Official
   - Make sure that only Given Name is required

#. Edit the Email attribute

   - Set :guilabel:`Description` to "Please use your institutional (university, research institution) email address if possible"
   - Change its attribute definition to Preferred rather than Official

#. Set :guilabel:`Submission Redirect URL` to ``https://<environment>/enrollment/thanks-for-signing-up``.
   ``<environment>`` should be the domain name of the corresponding Phalanx environment.

#. Set :guilabel:`Confirmation Redirect URL` to ``https://<environment>/enrollment/thanks-for-verifying``.

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

#. Go to the "Self Signup With Approval" enrollment flow
#. Attach the IdentifierEnroller as a wedge
#. Select :guilabel:`Configure`
#. Select :guilabel:`Manage Identifier Enroller Identifiers`
#. Select :guilabel:`Add Identifier Enroller Identifier`
#. Set the label to ``Username``
#. Set the description to "The username you will use inside the Rubin Science Platform. Must start with a lowercase letter and consist of lowercase letters, numbers, or dash (-)"
#. Set the identifier type to ``UID``

Finally, to work around multiple bugs in the enrollment process, we use a `a custom plugin <https://github.com/cilogon/Lsst01Enroller>`__.
This does the following:

- Find any group that a user was added to as part of the petition and reprovision those groups.

- Sets the preferred name chosen by the user during enrollment as primary, ensuring that the name chosen by the user overrides what comes from their identity provider and working around the fact that CILogon doesn't get names from GitHub and thus by default shows GitHub identities with opaque identifiers.

- Handles accidentally abandoned enrollment flows by returning the user to where they left off.
  Specifically, via a hook in the start step, if the login identifier exists and is linked to an Org Identity that is linked to a CO Person record in the "Pending Confirmation" state and that has no ``Name`` object attached of type "Preferred" and no email address attached of type "Preferred," the user is redirected back to the prompt for enrollment attributes, rather than trying to create a new stub person record (which would fail).

- If there is an existing Org Identity with login Identifier, CO Person record in the Pending Confirmation state, and a ``Name`` object and email address of type "Preferred," diagnose this as an enrollment still waiting email confirmation.
  Rather than trying to create a new person record, redirect the user to a configurable URL that tells them that their petition is awaiting email confirmation.

- If there is an existing Org Identity with login Identifier, CO Person record in the Pending Approval state, and a ``Name`` object and email address of type "Preferred," diagnose this as an enrollment waiting for approval.
  Rather than trying to create a new person record, redirect the user to a configurable URL that explains the approval process.

As with the identifier enroller plugin, this Lsst01Enroller plugin is installed as a wedge.
Because this requires installation of a custom plugin, this is done by the CILogon administrators rather than by Rubin Observatory staff.

To configure the two URLs used in the last two checks:

.. rst-class:: compact

#. Go to the "Self Signup With Approval" enrollment flow
#. Select :guilabel:`Attach Enrollment Flow Wedges` (top right)
#. Select :guilabel:`Configure` for the Lsst01Enroller plugin
#. Set the pending approval link to ``https://<environment>/enrollment/pending-approval``
#. Set the pending confirmation link to ``https://<environment>/enrollment/pending-confirmation``

These are currently placeholder pages that we need to customize, and may move elsewhere once we have customized them.

Configure names
---------------

The default name configuration adds a field for an honorific, which is not useful to us.

Ideally we would represent all names as a simple text box that allows the user to enter an opaque string, but unfortunately the COmanage data model requires separating the name into components.
The best compromise available between letting someone enter as much of their name as they wish and not prompting for too much spurious data is to configure the name fields as given and family only.
Users can enter values with spaces, commas, etc. in those fields if needed.

#. Go to :menuselection:`Configuration --> CO Settings`
#. Change :guilabel:`Name Required Fields` to "Given Name"
#. Change :guilabel:`Name Permitted Fields` to "Given, Family"

Self-service attribute changes
------------------------------

We want users to be able to change their name and email address whenever they wish.

.. rst-class:: compact

#. Go to :menuselection:`Configuration --> Self Service Permissions`
#. Select :guilabel:`Add Self Service Permission`:

   - Model: ``Name``
   - Type: ``Preferred``
   - Permission: ``Read Write``

#. Select :guilabel:`Add Self Service Permission`:

   - Model: ``Email Address``
   - Type: ``Preferred``
   - Permission: ``Read Write``

Username validation
-------------------

Ensure the `Regex Identifier Validator Plugin`_ is enabled.  Then:

.. rst-class:: compact

#. Go to :menuselection:`Configuration --> Identifier Validators` and add a new validator
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

The plugin used is `GroupNameValidator <https://github.com/cilogon/GroupNameValidator>`__ with the following configuration::

    Configure::write('GroupNameValidator.pattern', '/^g_[a-z0-9._-]{1,30}$/');
    Configure::write('GroupNameValidator.flash_error_text', 'Name must start with g_ and use only a-z,0-9,.,_, and -');

Email
-----

Change the template for the email message sent to users to prompt them to confirm their email:

#. Go to :menuselection:`Configuration --> Message Templates`
#. Select :guilabel:`Edit` for Self Signup with Approval Enrollment Flow Verification
#. Change the message subject to::

       Please confirm your Rubin Science Platform registration

#. Leave the format as plain text and change the message body to the following.
   Kkeep the hard wrapping, but change the URL of the environment at the end of the second paragraph:

   .. literalinclude:: email-confirm.txt

Then, email the COmanage support folks and ask the outgoing mail messages to use a ``Reply-To`` header of ``rsp-registration@lists.lsst.org``.
(This change can't be made through the configuration interface.)

Navigation links
----------------

Add a link to the corresponding Science Platform instance to the top bar of the COmanage interface:

.. rst-class:: compact

#. Go to :menuselection:`Configuration --> CO Navigation Links`
#. Select :guilabel:`Add CO Navigation Link`

   - Description: ``Link to corresponding Science Platform instance``
   - Link title: ``Science Platform (INT)`` (changing or removing the part in parentheses)
   - Link URL: URL of the Science Platform instance

Other helpful links (such as to documentation for how to use COmanage once we have it) can be added similarly.
See `Configuring Navigation Links <https://spaces.at.internet2.edu/display/COmanage/Configuring+Navigation+Links>`__ for more details.

Timeouts
========

When the user goes to COmanage (to manage their groups or identities, for example) and are redirected to CILogon to authenticate, they have 15 minutes to complete the authentication.
If they take longer than that to complete their authentication, they will receive a Timeout red error message after returning to COmanage and will have to start again.

Once a user has authenticated to COmanage, the session timeout is eight hours, after which they'll be logged out and will have to log in again.

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

Use with Gafaelfawr
===================

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

- COmanage can be themed following the instructions at `Theming COmanage Registry <https://spaces.at.internet2.edu/display/COmanage/Theming+COmanage+Registry>`__.
  We haven't yet looked in detail at this, let alone started.
  Any required custom CSS, JavaScript, or images will need to be uploaded to the server by the CILogon administrators before we can use it.

- COmanage comes with a bunch of default components that we don't want to use (announcement feeds, forums, etc.).
  Edit the default dashboard to remove those widgets and replace them with widges for group management and personal identity management (if there are any applicable ones).
