# Conversation on [SEG-Y blog post](https://agilescientific.com/blog/2014/3/26/what-is-seg-y.html), 26 March 2014

## Toastar 3 years ago 

The problem with segy is it's really good at what it designed for. It is meant to make transferring data between systems easy. SEGY rev 2 isn't going to change much. your software still needs to be able to read in data on 8mm tapes. Which means it is designed to meet the lowest common denominator rev.0. yes it makes for some odd scenarios like not being sure if your data is ieee float or 8 bit int. but that was always what segy was designed for, to be an archival standard.

What we need is a standard internal data format. right now everyone does everything differently. why do we need different formats? 3dv, 2v2, ksd, 1, cvbs, zgy, sgy with sort tables.

I think I'm sold on HDF5 being the solution to this. The basic concept is similar to rsf format, but with much more flexible headers. plus the ability to write out b-tree sort indices.

The downside to this approach is you end up with larger files, but I think it's worth it for the performance gain. But the gains I think are worth it. I never understood why sort tables never made it into interpretation software. They are so critical to processing workflows, but did developers ever think interpreters wouldn't want to be able to pull up x-lines, arb-lines, and time-slices quickly?

/rant

## Maitri 3 years ago 

I'd argue that 181 185 189 and 193 are also Strongly Recommended, as they are critical for loading and positioning the data in any software from Petrel to GOCAD.

Also, I hereby second Toastar's HDF5. Interpreters have better use of their time than wading through .3dv, .cmp, .bri, .zgy and .2v2 files today and waiting while an arb line or time slice loads (to see if the data is really there). Had to do it just today.

## Matt Hall 3 years ago 

@Toastar: Love the rants! I keep hearing about HDF5, and clearly have some catching up to do. I always quite liked the chunking and compression employed by Landmark's .bri format, and see that HDF5 supports this sort of approach. With modelr we're thinking more and more about transmitting lots of seismic data very quickly over the Internet, which is challenging us to do better than "send lots of JPEGs".

@maitri: In the next post, maybe tomorrow, I'll argue that I'd rather not have position data in the headers — except maybe when the data are fresh out of the oven. I'd rather get cornerpoints from a 'purer' source like the processing report (say), and set up my survey independently of the SEG-Y, then load the traces in by line/xline address. In my experience, it's not so much the wrongness of the coordinates, as the wrongness or missing-completely-ness of the coordinate system, datum, or units of measurement.

## Chris LeBlanc 3 years ago 

Excellent post Matt. I'll add what I've learned over the last couple years working with HDF5 and seismic processing.

I've been pushing the use of HDF5 as our internal format in GLOBE Claritas seismic processing software. We used a variant of SEG-Y as our internal format, but it was getting more and more painful to add new features in a backward compatible way.

One decision we made was to emulate SEG-Y rev 0 in HDF5, to make the transition as painless as possible. We've used the same file headers and trace headers that are defined in SEG-Y, but we also allow the use of additional headers and "pseudotraces" (eg: a seismic attribute). One of the major benefits of HDF5 is that it's self-describing, so every "object" in the file has metadata associated with it. We use a 2D HDF5 dataset to store our trace data, and this has metadata describing the number of samples, traces, datatype, dimensionality, compression, etc. Supporting the headers from revision 2 would be pretty easy.

The HDF5 files we produce are only slightly larger than their SEG-Y counterparts. A quick test I've run shows it's roughly 5% larger for 4MB files, and only 3% larger for 2GB (2184167144 bytes vs 2122003600 bytes). We're happy with a file that's only 3% bigger. The key is getting HDF5 chunk values that minimize wasted space. This is for uncompressed data. Depending on the compression type, you could see a drastic reduction in file size with HDF5.

Performance is also quite good. Our previous internal format was roughly the same speed as "dd" in Linux, so close to the maximum I/O available from the drive. With a bit of tweaking we've got our HDF5 code to run virtually as fast for sequential I/O (within a couple percent). We've even seen some non-sequential access run quicker with HDF5 than with our SEG-Y implementation.

Matt mentioned that reading SEG-Y is not for the squeamish. That is definitely true, especially if you want to have a single reader/writer that can handle all of these revisions. It's even worse if you want to support the many non-standard variants of SEG-Y. HDF5 makes this much simpler because every object describes exactly what it contains, there is no mystery about the endian type, datatype, dimensions, etc.

I have a demo Python program that loads trace headers and data from a file in roughly 4 lines of code using h5py. This opens up a new world of possibilities for people that want to manipulate their files without needing to parse an undocumented format. HDF5 removes vendor lock-in, at least in terms of accessing your seismic data.

Here's a quick list of the advantages of HDF5, off the top of my head:

- self describing, portable, high performance
- active open source community, tools, mailing list, optional commercial support from the HDF group
- already well established in many scientific fields, including demanding HPC fields like astronomy
- excellent langauage support: C, C++, Java, Python, MatLab, etc. Python especially has great support with PyTables and H5Py
- support for several compressions schemes, including custom compression methods
- n-bit compression and scale-offset compression
- chunking (aka bricking) support
- parallel I/O features for reading and writing with MPI (we haven't used these yet).

At this stage we're only using HDF5 as our internal format, we're not recommending it as a formal industry standard. However, I've returned from an exhausting trip around the world where I spoke to several big names in seismic processing and interpretation software. We're planning on collaborating on an open source project that would define an HDF5 standard for seismic files, and would also host the reference implementation for reading/writing. We're still in the early stages so I don't want to drop names, but it holds a lot of promise. I think we can have a powerful modern format that will address the needs of both HPC for processing and interpretation/visualization.

## Kelsey 3 years ago 

3-byte sample format? I assume that's fixed point. In what situation is 2 bytes not enough precision, but I really can't spring for 4 bytes?

## Bert 3 years ago · 1 like

Sigh. More work, more options, more ways for people to screw up SEG-Y files. That is what happens in practice.

More fields that will never be used, more confusion. Extra trace headers! I mean how much of the trace header is actually used?

This is a bad development. Data descriptions in a time that we all know how it should be done: compare PNG or JPEG. What good is it to describe data formats if you can define interfaces to access the data? And free libraries that everybody simply links?

Lastly: it seems SEG-Y is headed to become a vehicle to transfer entire databases. Well, most people are happy if at least they can get out the sample values with some trace keys and reasonable coordinates.

Can I download this new standard somewher? Need to allocate time to write code that defends against all these possible 'enhancements'.

## Toastar 3 years ago 

Oh here is the draft.

http://www.seg.org/documents/51956/6062543/SEG+Y+rev+2+Draft+June+2013?version=1.0

## Scott King 3 years ago 

SEG-Y rev1 has been out for over 10 years but it seems there is a very limited use of it. When I worked in software development we put in support for rev 1 many years ago in anticipation of a huge demand for it but it never seemed to come. Any thoughts on that and if you think rev 2 will be adopted more successfully than rev 1 seemed to be?

## Ben Rimmer 3 years ago 

Matt and Toastar, the latest draft is at the following link, it's from February 2014:

http://www.seg.org/documents/51956/6062543/SEGY+2.0+draft+February+2014

## Matt Hall 3 years ago 

@Ben: Thank you for posting the latest on the new specification. Do you know what the timeline on its approval and adoption might be? Are people already using it? I'm also curious how many people responded to the call for input I heard about at SEG.

On a sort-of-related note, I was chatting to someone over email about corporate seismic databases. They are typically a rather scary mess. The old the company (and the more acquisitions and mergers it has been through) and the more explored the basin, the worse it is. Much paper, many tapes, many directories of undocumented projects, many layers of reprocessed and reinterpreted data... Added to which is the complexity of seismic data licensing. There's plenty of room in SEG-Y files for enough metadata to sort a lot of this mess out, but... well, clearly the system is badly broken. I doubt SEG-Y Rev 2.0 can have much impact on this sort of mess, unfortunately. It's definitely a cope don't fix situation, but we nowhere near coping with it.

## Bert Bril 2 years ago

@Scott: Actually, Rev.1 is by now widely adopted (the major exception being SMT/(Kingdom Suite). This is because using Rev. 1 really adds something that everyone needs: clarity about where to find inline/crossline and CDP coordinates. That's all. Nobody uses the extra fluff like extended headers.

@Matt: I'm sorry to have to say this but IMO Rev.2 is a step back. More options means there are more ways in which implementations can go wrong and interpretations can vary. Most developers simply ignore the new possibilities except that they will need to defend against possible hazards like varying trace lengths and extra trace headers. And still try to extract what is need from the file: fixed traces of samples at certain positions

If you say: 'yeah, but where are these positions then?', well, the whole idea of defining the precise coordinate system in the SEG-Y file is a no-go in practice anyway. These 'stanzas' are never present, so nobody can rely on them. That's why you define that in another place, not in the SEG-Y file. Which is good. We are exchanging seismic data, not entire databases.

This is the crux: do not burden all who have to read and write seismic data with companies' lack of a good database around their data. That will not be fixed by SEG-Y, and shouldn't be, either. Take a look at initiatives like PetroBank in Norway. They made a database around managed SEG-Y (and other) files.

Anyway, the future is in interfaces, not data format specifications. Give me services to access the data and I don't care how it's stored ...

## Jill Lewis 2 years ago 

I hope that you are all members of the Technical Standards of the SEG, and you cannot use not being a member as an excuse it's open to all.

Facts

- Some years ago standard's names were standardized as SEGD, SEGD1, SEGD2, SEGD2.1, SEGD3, SEGY, SEGY1 and SEGY2.
- The HDF5 group and the SEG already have an agreement and the EMCS group have been discussing using this as a base for their exchange format.
- SEGY2 does not have high recommended fields but mandatory fields.
- What would be the volume of a post-stack 990 Gbyte (yes we have seen one) SEGY dataset if written as HDF5 data out of interest?
- The TSC has looked at xml etc in the past.

## Brad Pepers A year ago 

Your table of binary header values shows 117 as in ms but it's actually in micro-seconds for time but other things for frequency or depth.

Does anyone know what it is for depth seismic? Is it feet or milli-feet (yes let's pretend that's a real thing!)? So for example if the depth seismic has a 2 foot sample interval would you find "2" or "2000" at offset 117 in the binary header? Does anyone ever do fractional foot/meter sample intervals?

## Tom Popowski A year ago 

I second this request. As far as I can tell, neither Rev0 nor Rev1 make any allowance for a sample interval in any units other than µs. I usually see a comment in the text header that it is "depth." Since the binary header field (17-18) and trace (117-118) are 2-byte signed(!?) integers, the maximum value allowed is 32767 µs, or 32.767 ms.
