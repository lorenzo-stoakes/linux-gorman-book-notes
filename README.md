# Understanding the Linux Virtual Memory Manager

This is a set of notes I am amateurishly extracting from the book
[Understanding the Linux Virtual Memory Manager][amazon] written by the esteemed
[Mel Gorman][mel].

These are written for my own tastes and understanding so will likely not be
useful for anybody else, but I am placing them here primarily as a record for
myself, though if others find them somehow useful (2.4.22 fetishists perhaps? ;)
then that's great too!

[The book][book] is licensed under an [Open Publication License][license] with
the options "no substantial derivitives" and "no distribution for commercial
purposes".

Though this is a (substantial) derivative, it is entirely without commercial aim
of course, and designed merely as a study guide for myself.

__NOTE:__ Whenever there is an architecture-dependent function/macro/data
structure, the links I am including reference i386.

__NOTE:__ For the time being, I am skipping the 'What's new in 2.6' parts of the
book as I am focused on 2.4.22. Looking through these will be part of the next
phase of my research where I translate from 2.4.22 to a recent kernel.

__NOTE__: I skip chapter 1 as this is introductory and doesn't contain many
details of interest to me at this point.

## Contents

* [2: Describing Physical Memory](2.md)

* [3: Page Table Management](3.md)

* [4: Process Address Space](4.md)

* [5: Boot Memory Allocator](5.md)

### Incomplete

* [6: Physical Page Allocation](6.md)

### Non-Existent

* [7: Non-contiguous Memory Allocation](7.md)

* [8: Slab Allocator](8.md)

* [9: High Memory Management](9.md)

* [10: Page Frame Reclamation](10.md)

* [11: Swap Management](11.md)

* [12: Shared Memory Virtual Filesystem](12.md)

* [13: Out of Memory Management](13.md)

[amazon]:http://www.amazon.co.uk/Understanding-Virtual-Memory-Manager-Perens/dp/0131453483
[mel]:http://www.csn.ul.ie/~mel/blog/
[book]:https://www.kernel.org/doc/gorman/
[license]:https://www.kernel.org/doc/gorman/license.html
