How to write and build the documentation
========================================

Requirements
------------

We use Restructured Text (RST_) and Sphinx_ to build this document.

There is a nice short tutorial_ that goes into details of how things
are setup and work together.

The template used currently is the cloud_sptheme_

Installation is left up to the reader. **Note**: In order to *contribute writing*, 
you don't need to actually install any software. A texteditor is all you will need
(or you can edit directly in the WebGUI of SWITCHdrive).

A note on installation:

On Mac OSX (Mavericks):
Install Python (with homebrew)

   # install / update easy_install
   curl https://bootstrap.pypa.io/ez_setup.py -o - | python   

   # install sphinx
   easy_install -U sphinx

   # install cloudtheme
   easy_install cloud_sptheme


There are two main directories: ``source`` and ``build``. ``source`` contains the 
actual text, ``build`` contains the compiled documentation in various
formats.

Writing documentation
---------------------

The document is structured into different chapters. The file ``index.txt`` ties
together the chapters.

As we are sharing the writing process via SWITCHdrive (an ownCloud installation - 
we live the *eat your own dogfood*) we need to be a bit careful when editing text.
In an ideal world, simultaneous changes would be detected automatically - but alas -
we don't live in an ideal world.

Using version control would be a possibility, but probably too complicated for
most casual editors. ownCloud sometimes detects conflicts and flags them with
a conflict file, but this is not bulletproof.

Therefore, we revert back to the *ways of the good old days* and lock the files
we are editing.

**Lock a file** by renaming it - for example: ``architecture_xyz.txt`` where ``xyz``
are your initials. When you are done editing, revert the filename.

Building documentation
----------------------

If you have the necessary software installed, just run ``make`` to see a list
of formats that you can build and then run one of them::

    make html

The output files will be in the ``build`` directory.





.. 
  _cloud_sptheme: https://pythonhosted.org/cloud_sptheme
  _RST: http://docutils.sourceforge.net/rst.html
  _Sphinx: http://sphinx-doc.org/tutorial.html
  _tutorial: http://tompurl.com/2012/11/22/writing-a-book-with-vim-restructured-text-and-sphinx/
