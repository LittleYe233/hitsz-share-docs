API Reference
=============

This is the API reference of the whole project, including function definitions, parameter notes and so on.

.. note::

   Most of the project is written in JavaScript, and declarations of many functions are placed in ``.d.ts`` files, not enclosed in the ``.js`` script files like JSDoc. So please remember that there are many definitions of ``type``\s that may be confusing and not clear when you use an IDE to check the types of some identifiers. Certainly, details of all the ``type``\s are shown in this reference. And if you have just followed a link from the ``.js`` code - congratulations! You find the right place!

.. note::

   "Path" fields in the reference documents are hyperlinks to the corresponding files on the *default* branch - usually ``main`` which isn't always the latest - on GitHub. See also the files with the same name on a *developing* branch - usually ``dev``.

.. note::

   Notes on ``.d.ts`` files are placed in the documents of the corresponding ``.js`` files.

.. note::

   Python module ``docutils`` doesn't support directive entries for TypeScript now. ``type``\s in TypeScript are regarded as ``js:class``\es.

.. toctree::
   :maxdepth: 4

   tracker/index.rst