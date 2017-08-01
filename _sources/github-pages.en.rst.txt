.. include:: common.defs

Setting Up Github Pages
***********************

Github can provide simple web hosting based on repositories. This seems to be available from both of
our internal github instances (`Corporate Git <http://git.corp.yahoo.com>`_ and `Partner Git
<http://partner.git.corp.yahoo.com>`_) and from the |TS| open source host `Github
<http://github.com>`_.

This document does not explain the full power of `Github Pages <http://pages.github.com>`_ - click
on the link for the full details. What is explained here is a specific application of Github Pages
for Edge Group developers.

The goal of this is to have a repository containing documentation in
`Sphinx <http://sphinx-docs.org>`_ format which can be published to Github Pages, thereby avoiding
the need to have a LAMP server which must be allocated and maintained. In the setup I use I have two
repositories. One contains the documentation source and support files. The other is used only to
contain the HTML generated from the documentation source. Sphinx was chosen because that is the
documentation engine for |TS| and therefore all Edge Group |TS| developer should be familiar with it.
If not, learn it.

My repositories are the `source repository <https://git.corp.yahoo.com/solidwallofcode/docs>`_ and
the `publish repository <https://git.corp.yahoo.com/solidwallofcode/solidwallofcode.github.io>`_.
The source can have any name but the publish must be named in the pattern `username.github.io` where
``username`` is replaced by your Github user name (which should be the same as your Oath: login name).

In my source repository is a makefile which performs the two basic operations, generation and
publication. Generation done by :code:`make html` which runs Sphinx on the documentation source,
generating HTML output in the :file:`docbuild/html` directory. It is by design that changes and updates to
the source can be committed and pushed independently of publishing so that intermediate results can
be saved. When the documentation is ready for a publishing update I use :code:`make publish`. This
assumes the publish repository has been cloned in a parallel directory named `swoc-publish`. It runs
the `publish.sh` script which clears out that directory, copies the contents of :file:`docbuild/html`,
commits it, and then pushes. Once the new content has been pushed it is served at the URL
`<https://git.corp.yahoo.com/pages/solidwallofcode/solidwallofcode.github.io>`_. For someone else,
the ``solidwallofcode`` will be replaced by your github username.

To set this up for yourself, I would

#. download (not fork) my source repository
#. unpack the download, clean out my documentation except for `index.rst`, and use it to create a repository.
#. create the publish repository and clone it locally.
#. adjust :file:`publish.sh` to use the directory where your publish repository is cloned.
#. Generate the documentation (which should be just a shell from `index.rst`) and publish it.
#. See if your pages are live.

.. note::

   Github Pages uses a HTML engine called `Jekyll <https://help.github.com/articles/about-github-pages-and-jekyll/>`_. By default this will *not* include subdirectories. To work around this you can add the :file:`.nojekyll` file to the root of your publish repository. This disables all pre-processing by Githug Pages. In my Sphinx :file:`conf.py` I have added the `Sphinx Github Pages extension <http://www.sphinx-doc.org/en/1.5.1/ext/githubpages.html>`_ to handle this.

.. note::

   Publishing is not always instaneous so it may be a few minutes before updated pages are available.
