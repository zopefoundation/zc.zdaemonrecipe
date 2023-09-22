**Caution!**

This repository has been archived. If you want to work on it please open a ticket in https://github.com/zopefoundation/meta/issues requesting its unarchival.

**************
zdaemon recipe
**************

The zdaemon recipe provides support for generating zdaemon-based run
scripts.

.. contents::

Releases
********

0.3.2 (unreleased)
==================

- Add support for Python 3.5 up to 3.8.

- Drop support for Python 2.6, 3.2, 3.3 and 3.4.


0.3.1 (2013-04-01)
==================

- Add MANIFEST.in, necessary with the move to git.


0.3 (2013-04-01)
================

- Added ``shell-script`` setting.  When true, shell scripts that refer
  to a zdaemon script in the software installation are generated instead
  of Python scripts in the rc directory.


0.2 (2008-09-10)
================

- Added support for the deployment recipe ``name`` option.


0.1 (2008-05-01)
================

Initial release.
