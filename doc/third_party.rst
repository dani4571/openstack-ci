HOWTO: Third Party Testing
==========================

Overview
--------

Gerrit has an event stream which can be subscribed to, using this it is possible
to test commits against testing systems beyond those supplied by OpenStack's
Jenkins setup.  It is also possible for these systems to feed information back
into Gerrit and they can also leave non-gating votes on Gerrit review requests.

An example of one such system is `Smokestack <http://smokestack.openstack.org/>`_.
Smokestack reads the Gerrit event stream and runs it's own tests on the commits.
If one of the tests fails it will publish information and links to the failure
on the review in Gerrit.

Reading the Event Stream
------------------------

It is possible to use ssh to connect to ``review.openstack.org`` on port 29418
with your ssh key if you are signed up as an OpenStack developer on Launchpad.

This will give you a real-time JSON stream of events happening inside Gerrit.
For example:

.. code-block:: bash

   $ ssh -p 29418 review.example.com gerrit stream-events

Will give a stream with an output like this (line breaks and indentation added
in this document for readability, the read JSON will be all one line per event):

.. code-block:: javascript

   {"type":"comment-added","change":
     {"project":"openstack/keystone","branch":"stable/essex","topic":"bug/969088","id":"I18ae38af62b4c2b2423e20e436611fc30f844ae1","number":"7385","subject":"Make import_nova_auth only create roles which don\u0027t already exist","owner":
       {"name":"Chuck Short","email":"chuck.short@canonical.com","username":"zulcss"},"url":"https://review.openstack.org/7385"},
     "patchSet":
       {"number":"1","revision":"aff45d69a73033241531f5e3542a8d1782ddd859","ref":"refs/changes/85/7385/1","uploader":
         {"name":"Chuck Short","email":"chuck.short@canonical.com","username":"zulcss"},
       "createdOn":1337002189},
     "author":
       {"name":"Mark McLoughlin","email":"markmc@redhat.com","username":"markmc"},
     "approvals":
       [{"type":"CRVW","description":"Code Review","value":"2"},{"type":"APRV","description":"Approved","value":"0"}],
   "comment":"Hmm, I actually thought this was in Essex already.\n\nIt\u0027s a pretty annoying little issue for folks migrating for nova auth. Fix is small and pretty safe. Good choice for backporting"}

For most purposes you will want to trigger on ``patchset-created`` for when a
new patchset has been uploaded.

Further documentation on how to use the events stream can be found in `Gerrit's stream event documentation page <http://gerrit-documentation.googlecode.com/svn/Documentation/2.3/cmd-stream-events.html>`_.

Posting Result To Gerrit
------------------------

External testing systems can give non-gating votes to Gerrit by means of a -1/+1
verify vote.  OpenStack Jenkins has extra permissions to give a +2/-2 verify
vote which is gating.  Comments should also be provided to explain what kind of
test failed..  We do also ask that the comments contain public links to the
failure so that the developer can see what caused the failure.

An example of how to post this is as follows:

.. code-block:: bash

   $ ssh -p 29418 review.example.com gerrit review -m '"Test failed on MegaTestSystem <http://megatestsystem.org/tests/1234>"' --verified=-1 c0ff33

In this example ``c0ff33`` is the commit ID for the review.  You can set the
verified to either `-1` or `+1` depending on whether or not it passed the tests.

Further documentation on the `review` command in Gerrit can be found in the `Gerrit review documentation page <http://gerrit-documentation.googlecode.com/svn/Documentation/2.3/cmd-review.html>`_.

We do suggest cautious testing of these systems and have a development Gerrit
setup to test on if required.  In SmokeStack's case all failures are manually
reviewed before getting pushed to OpenStack, whilst this may no scale it is
advisable during initial testing of the setup.

.. _request-account-label:

Requesting a Service Account
----------------------------

To request a sevice acconut for your system you first need to create a new
account in LaunchPad.  This account needs to be joined to the
`OpenStack Team <https://launchpad.net/~openstack>`_ or one of the related teams
so that Gerrit can pick it up.  You can then contact the
OpenStack CI Admins via `email <mailto:openstack-ci-admins@lists.launchpad.net>`_
or the #openstack-infra IRC channel.  We will set things up on Gerrit to
receive your system's votes.

Feel free to contact the CI team to arrange setting up a dedicated user so your
system can post reviews up using a system name rather than your user name.

The Jenkins Gerrit Trigger Plugin Way
-------------------------------------

There is a Gerrit Trigger plugin for Jenkins which automates all of the
processes described in this document.  So if your testing system is Jenkins
based you can use it to simplify things.  You will still need an account to do
this as described in the :ref:`request-account-label` section above.

The OpenStack version of the Gerrit Trigger plugin for Jenkins can be found on
`the Jenkins packaging job <https://jenkins.openstack.org/view/All/job/gerrit-trigger-plugin-package/lastSuccessfulBuild/artifact/gerrithudsontrigger/target/gerrit-trigger.hpi>`_ for it.  You can install it using the Advanced tab in the
Jenkins Plugin Manager.

Once installed Jenkins will have a new `Gerrit Trigger` option in the `Manage
Jenkins` menu.  This should be given the following options::

  Hostname: review.openstack.org
  Frontend URL: https://review.openstack.org/
  SSH Port: 29418
  Username: (the Launchpad user)
  SSH Key File: (path to the user SSH key)

  Verify
  ------
  Started: 0
  Successful: 1
  Failed: -1
  Unstable: 0

  Code Review
  -----------
  Started: 0
  Successful: 0
  Failed: 0
  Unstable: 0

  (under Advanced Button):

  Stated: (blank)
  Successful: gerrit approve <CHANGE>,<PATCHSET> --message 'Build Successful <BUILDS_STATS>' --verified <VERIFIED> --code-review <CODE_REVIEW> --submit
  Failed: gerrit approve <CHANGE>,<PATCHSET> --message 'Build Failed <BUILDS_STATS>' --verified <VERIFIED> --code-review <CODE_REVIEW>
  Unstable: gerrit approve <CHANGE>,<PATCHSET> --message 'Build Unstable <BUILDS_STATS>' --verified <VERIFIED> --code-review <CODE_REVIEW>

Note that it is useful to include something in the messages about what testing
system is supplying these messages.

When creating jobs in Jenkins you will have the option to add triggers.  You
should configure as follows::

  Trigger on Patchset Uploaded: ticked
  (the rest unticked)
  
  Type: Plain
  Pattern: openstack/project-name (where project-name is the name of the project)
  Branches:
    Type: Path
    Pattern: **

This job will now automatically trigger when a new patchset is uploaded and will
report the results to Gerrit automatically.

