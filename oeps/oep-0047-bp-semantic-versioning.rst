###########################
OEP-47: Semantic Versioning
###########################

.. list-table::

   * - OEP
     - :doc:`OEP-0047 </oeps/oep-0047>`
   * - Title
     - Semantic Versioning
   * - Last Modified
     - 2020-05-08
   * - Authors
     - Robert Raposa <rraposa@edx.org>
   * - Arbiter
     - ???
   * - Status
     - Draft
   * - Type
     - Best Practice
   * - Created
     - 2020-03-19
   * - Review Period
     - ???

.. contents::
   :local:
   :depth: 3

Summary
=======

For any Open edX repository that publishes versioned packages, this OEP recommends following the `Semantic Versioning v2`_.

The most relevant part from the summary of the `Semantic Versioning v2`_ specification:

    Given a version number MAJOR.MINOR.PATCH, increment the:

    * *MAJOR* version when you make *incompatible API changes*,
    * *MINOR* version when you *add functionality in a backwards compatible manner*, and
    * *PATCH* version when you make *backwards compatible bug fixes*.

Some additional tips are called out in `Backward Incompatible Changes`_ and `Tooling and Automation`_.

.. note::

    This OEP does not apply to repositories without versions. For example, services like edx-platform, ecommerce, etc. do not have versions.

Context
=======

Many or all of the Open edX installable libraries already use versions similar to the `Semantic Versioning v2`_ specification. This OEP simply codifies this best practice, and provides additional guidance.

Backward Incompatible Changes
=============================

Why do incompatible changes need special attention?
--------------------------------------------------

* Communicating incompatibilities is important because it alerts humans to possible problems when attempting to upgrade your package.
* A MAJOR version is used to communicate any "Backward Incompatible Changes" (also known as "Breaking Changes").
* If you think a breaking change is unlikely to affect anyone, increment the MAJOR version anyway, and simply note that fact in the changelog entry.

What if I find an incompatible change after it is released as a MINOR or PATCH version?
---------------------------------------------------------------------------------------

If you later find a backward incompatible change in a MINOR or PATCH version after publishing the package, please update the changelog and communicate the inconsistency. Additionally, you could do one of the following:

* A new MINOR or PATCH version could be released to revert the change, and the change can then be restored with a new MAJOR version. Or,
* If you decide that reverting will cause more harm for some reason, and you do not intend to revert, please let this be a part of your communication.


Tooling and Automation
======================

* A checklist item in a pull-request template can remind people of any manual steps.
* Some frontend libraries have been using an npm package called `semantic-release`_. The `semantic-release`_ package automates incrementing the version appropriately for each PR merge, based on a well formatted commit comment.

  * One important formatting detail is to include "BREAKING CHANGE" at the beginning of a line in the commit description to get a new major version.

* Other tools can be added here over time.

.. _semantic-release: https://github.com/semantic-release/semantic-release

References
==========

* `Semantic Versioning v2`_

.. _Semantic Versioning v2: https://semver.org/spec/v2.0.0.html
