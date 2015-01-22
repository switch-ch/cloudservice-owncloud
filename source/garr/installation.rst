GARR Installation
=================

Prerequisites
-------------

Software
^^^^^^^^

GARR is currently evaluating Fabric_ as a supporting tool for system administration.

Fabric is a Python (2.5-2.7) library and command-line tool for streamlining the use 
of SSH for application deployment or systems administration tasks.
It provides a basic suite of operations for executing local or remote shell commands 
(normally or via sudo) and uploading/downloading files, as well as auxiliary 
functionality such as prompting the running user for input, or aborting execution.
Fabric is logically divided in two phases: configuration and deploy where in 
configuration phase we prepare the garrbox installation and we run a deploy phase 
for every production machine.



.. links

.. _Fabric: http://www.fabfile.org/
