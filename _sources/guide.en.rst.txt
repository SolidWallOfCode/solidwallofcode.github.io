.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _guides:

Open Source Contributions
*************************

Contributing to open source can be a struggle given the variety of concerns from the community.
Overall, however, my view is that code is made stronger by surviving the gauntlet. There are many
skilled developers in the community who will often see things you miss.

The community has undergone rather a lot of growth lately and so is still adapting to an overall
group where it it not the case that every one knows every one else from chat and summits and there
is a shared understanding of what makes a good pull request. The following are I think the most basic
and important things to keep in mind that will help ease the process.

Separability
   Pull requests should be as small as possible. This yields the important result that isolating
   changes that cause problems and recovering from them is much easier. This means that if new
   classes / technology / support is required for a pull request, it is almost always better to
   contribute it in a separate, prior pull request. Pull requests that have a lot of new classes and
   other things have a very difficult time, both because the additional support obscures the real
   point of the pull request and that discussions can get bogged down on details in the support that
   aren't critical for the overall work.

Parsimony
   There is a general concern about code bloat and much effort has been made in the past to decrease
   the over all code size. For what |TS| can do it has a remarkably small code base. This is not an
   accident. For this reason think twice, then reconsider again, if you really need to add a support
   class. Avoid this if you can, especially if the class is used only once or in limited
   circumstances. Also avoid putting classes in separate files if the class isn't used in other file
   scopes. See if there are existing classes or other techniques which, with a bit of work, could be
   made to suffice.

Consistency
   |TS| is an old project with people who have been working on it for over a decade. They are
   comfortable with where things are and how things are done. This doesn't mean it's perfect but
   unless there is a strong reason to change, you should peruse the code and try to do things in a
   similar way. One example would be initialization. There are various ways to do this in C++, most
   of which result in the same machine code. Therefore it's a reasonable expectation that you will
   do this in the same way as the existing code. As a newcomer to the community, try to show a bit
   of respect for existing customs and habits, *especially* when such things have no real impact on
   the code quality.

Additional reference pages
   `Basic coding style <https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65146830>`_.

   `C++11 Usage Guide <https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65146830>`_.
