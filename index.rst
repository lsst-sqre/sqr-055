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

Dashboard
---------

COmanage comes with a bunch of default components that we probably don't want to use (announcement feeds, forums, etc.).
We will want to edit the default dashboard to remove those widges and replace them with widges for group management and personal identity management (if there are any applicable ones).

Integration
===========

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

User onboarding API
-------------------

When a new user first accesses the Rubin Science Platform, we will need to route them through the onboarding flow, and then may need to make additional changes to their record via the COmanage API.
For example, if the CILogon numeric identifier isn't usable as a numeric user ID, we may need to assign one.
This can be integrated with the onboarding service described in SQR-052_.
This service would have a privileged API token for the Rubin Science Platform COmanage environment.

.. _SQR-052: https://sqr-052.lsst.io/

Open questions
==============

#. What is the URL of the API service?

#. What is the hostname and authentication mechanism for the LDAP service?
   Is this the best way to retrieve client metadata?

#. Where can numeric UIDs and GIDs be stored in user and group metadata?
   Each user must have a unique numeric UID, which is also reserved as a GID.
   Each group must have a unique numeric GID that does not overlap with any UIDs.
   Assignment can be done via an integration service that looks for new users, but this information ideally should be stored in COmanage somewhere.
   Where should it be stored?
   None of the default identifier types look suitable for this purpose, but I can't find detailed documentation for all of them.
   Perhaps we should use the CILogon numeric ID?
   (This was the recommendation from our original requirements review.)

#. How to restrict inbound CILogon authentications to the Rubin Science Platform to only registered COmanage users?
   Is this functionality that's built into CILogon and COmanage in some way, or will the Rubin Science Platform authentication layer need to check COmanage to see if the user already exists and send them into an account creation flow if they do not?
