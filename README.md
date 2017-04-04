# xsdf
X Seismic Data Format

This is a set of rough notes about a fictional future format for the exchange of seismic reflection data.


## Principles

- 100% permissive open source everything.
- For discussion: no 3rd party libraries? (This would eliminate HDF5.)
- Implemented in code, not just words.
- Language and platform agnostic.
- No XML :)


## Components

- A specification document.
- A software library implementing reading and writing of the format. Probably in C.
- Wrappers in other languages (at least Java and Python).
- Translators to and from SEG-Y, SEG-D, and JavaSeis.
- Integrity checking code would also be a requirement.


## Feature requirements

- Compression.
- Parallel I/O.


## Stuff to read

- [Conversations from Software Underground](references/Software_Underground_chat.md)
- [Stack Exchange question](references/Stack_Exchange_question.md)
- [Blog post comments](references/Blog_post_comments.md)
- [Bert Bril's description](https://github.com/softwareunderground/xsdf/wiki/Old_specs_bert)
