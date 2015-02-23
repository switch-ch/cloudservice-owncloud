.. _cesnet-modifications:

CESNET Modifications to ownCloud
================================

We aren't running just a stock ownCloud source
with some 3rd party apps installed. During the
time we provided our sevice, we came across some
specific requirements, so we needed to develop
custom modifications. They are available at
our `cesnet/owncloud-apps`_ fork. This chapter
describes some of the modifications needed.

User WebDAV Authentication
--------------------------

In our setup, the 'user_saml' app is responsible for the task
of authenticating and managing the user accounts. Fixing it
as discussed in the :ref:`samlfix` section enabled users
to log in and automatically create accounts.

But there were still problems that needed to be solved. Automatical
account creation had a side effect. Since users are entering their
login credentials at their organizations IdP, ownCloud has no
way knowing the user's password. That way, the 'user_saml' app must
set a long enough random password for each new ownCloud user
automatically created.

Users don't need to know their ownCloud password as long as they don't
use some WebDAV client. But clients need the ownCloud password and this
renders them completely unusable. To solve this, we have modded
the 'user_saml' application so it adds extra fields
to each user's 'Personal Settings' page, so that they can configure
their clients properly.

.. image:: images/modifications/setWDpwd.png

Account Consolidation
---------------------

Since we are operating the service
with federated accounts, it could easily happen
that some users have multiple accounts at multiple organization IdPs.
Normally, we would create an independent ownCloud account for each organization's
account the user has. But with consolidation, it is possible to map
all user's IdP accounts to a single ownCloud account. So when the user
leaves one organization and his account there is closed, he can still access
the same ownCloud data using the second IdP's account he may have.

.. NOTE::
	This topic is still a 'work in progress' for us, as we have just
	a basic support for this feature implemented. We are going to
	collaborate with the Perun_ AAI developers and use Perun's
	consolidation mechanisms.


File Sharing
------------

There were several improvement requests to the ownCloud 6 sharing
mechanisms that we implemented ourselves (they were just too specific
to our environment to be considered suitable for a pull request to upstream).

By default, the share's autocomplete function offers users with usernames (and full names)
of the other users as they start typing. In our environment, this has
proven to be undesirable as our users cannot choose usernames. It is determined
by the EPPN identifier sent to us by the user's IdP. In some organizations,
an EPPN is considered to be a sensitive information, so we were asked
to disable the autocomplete function.

On the other hand, disabling autocomplete function would worsen the user experience when adding shares.
So we implemented an autocompletion from the user's contacts at least. You just need
to import contacts you want to quickly share files to in the 'Contacts' app.

We plan to integrate group management in ownCloud with our user management
system Perun. Perun defines groups and handles user membership in them, it
would be convenient for the users to be able to use the groups in all
systems we operate.

.. links:

.. _`cesnet/owncloud-apps`: https://github.com/CESNET/owncloud-apps
.. _Perun: https://github.com/CESNET/perun
