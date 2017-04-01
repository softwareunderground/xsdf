# Stack Exchange question, May 2014

[*What would a replacement for SEGY look like?*](http://earthscience.stackexchange.com/questions/694/what-would-a-replacement-for-segy-look-like)

I have been having a miserable time this week reading SEGY files. This is data from the largest seismic acquisition company in the world whose client is the 7th largest oil company in the world. So if anyone could get it right you would think it would be these two.

The SEGY standard for storing seismic data is used throughout the oil industry but is showing it's age quite badly. Problems include:

- non standard header information
- the textual header can be in ASCII or EBCDIC
- crazy machinations to save a few bytes
- binary data stored in IEEE or IBM floating point
- etc.

Some of the most glaring deficiencies are being patched up with a v2 release in the works. Details are here: http://www.seg.org/documents/51956/6062543/SEGY+2.0+draft+February+2014.

This is not a standard you can implement just by reading the spec and this new version won't help fix that. The continual abuse of the standard means that it takes calendar years to 'debug into existence' a library that can withstand real world data files. This is hugely frustrating for both the software developers and the geoscientists who have to work with it.

There was a discussion on this excellent blog recently where HDF5 was mentioned

- http://www.agilegeoscience.com/journal/2014/3/27/how-to-load-seg-y-data.html
- http://www.agilegeoscience.com/journal/2014/3/26/what-is-seg-y.html

I would prefer to see a new standard that is less complicated than SEGY and with no dependencies on third party libraries. So while I think technologies like HDF5 (and JavaSeis etc.) may have a role within a company they are not a replacement for SEGY. Even a technology like XML is going to be outlived by the data files we will have to read in the future. So I think the format will need to be extremely simple and conservative.

Some features of a format that could replace SEGY might include:

- one character encoding (ASCII or UTF-8)
- all headers in human readable form – key-value pairs?
- mandatory header information, e.g. shotpoints, x/y locations etc.
- standard unit definitions, map projections etc.
- traces may be binary but only IEEE
- data should be in one file (SEGY got this right)
- no data compression

So now for the question – what do we have to do to move beyond the extremely frustrating SEGY standard? Is there anything out there already we should just start evangelizing? If not, what do you think a replacement format needs to look like?

Thanks for reading. I feel much better after this rant and will get back to debugging why byte number 3603 is wrong.

EDIT: After chilling out over the weekend here are some more thoughts. SEGY is complicated for a number of reasons:

- inherent complexity
- accidental complexity
- different expectations for the format from users

So SEGY is hard to use because the seismic problem domain is an inherently difficult one to model. We are lucky to have the benefit of the wisdom of those who came up with the format.

Even so, a lot of cruft has accumulated in the format over the years. This creates a non-trivial cognitive burden that I could do without but will realistically just have to deal with.

The last point though is interesting. We already have SEGD for storing the raw field data. This format is a lot more challenging than SEGY but is also a good place to bury a lot of the inherent complexity of real world seismic data.

When people suggest HDF as a replacement I guess that (maybe) they are viewing SEGY as a 'processing' format?

Last week I was thinking of SEGY as a vehicle for sharing final seismic volumes with interpreters. I read at the weekend that 80% of SEGY headers are not populated in real world data. So, this is speculation on my part, but perhaps what we need is

- SEGD for raw field data
- something else for actually processing
- and something else again for sharing data between processors and interpreters

To accurately, and reproducibly, load seismic data into interpretation systems we don't need much meta-data but it absolutely does need to be present. The hand-off from processors to interpreters via SEGY is where I think we could use a new approach.

Candid Lunch

### COMMENTS

Well, the cheeky answer is that SEGY would be fine if it were actually a standard or a single file format. We just need to replace our thousands of different SEGY formats with some sort of standard. ...And yet another standard is born. :) – Joe Kington May 3 '14 at 1:53

I don't know SEGY or seismology, but how about something like netcdf? It's a binary format, true, but it's an open and "self-documenting" one, and seems to be gaining acceptance in some fields. – Simon W May 5 '14 at 21:03

It's a good question and SE might be a good place to discuss it. – Tbb May 14 '15 at 9:08 
add a comment


## ANSWER

I've done work with SEGY files, and the fixed length headers with ignored fields were an issue. Also, endian issues. You have to remember SEGY is the present exchange and archiving standard. That's a good thing.

The issue is that SEGY was designed for tape, so it's a single file. In this day and age, that may not be a good thing. It might be better to allow for extended 'metadata' to be housed as a separate file. Basically, the catalog record that you could review before buying the data. And you should be able to add this metadata back into the file.

If this was done as exchange profiles. Basically information groups rather than one large information block then a different format might get more buy in, and compliance over time.

As for HDF, and other formats, it's about how the data structures are laid out. There can be some performance issues with how timeseries data access (~traces) is organized in NetCDF files.

Some features of a format that could replace SEGY might include:

- one character encoding (ASCII or UTF-8) --- yes
- all headers in human readable form - key value pairs? --- yes
- mandatory header information eg shotpoints, xy locations etc. ---I would say yes, but I don't think that is going to happen.
- standard unit definitions, map projections etc. --- yes,
- traces may be binary but only IEEE
- data should be in one file (SEGY got this right) --- All data, yes. All information, no.
- single package like a tar file --- yes
- no data compression

Bits are cheap. Include details of the used dictionary items in the file. In this day and age, we don't need to have a translation book external to the file. But we should use standard reference terms that are online as Linked Open Data.

david valentine

### COMMENTS

I have had endian-ness issues with many binary formats but not SEGY which is big endian. The new SEGY rev 2 allows you to use little endian too which will make working with SEGY even more exciting. – Candid Lunch May 2 '14 at 16:03

I think one file is good though. For instance last week I downloaded the GIS files (many of them) for the Teapot Dome project. Did I get them all? I have no idea. I know I got the one segy file though. \ – Candid Lunch May 5 '14 at 20:30
add a comment


## ANSWER

The advantage of SEGY is also it's main problem; It's been around for very long. I've been struggling to open a decade old word document correctly, whiles one can actually still access a SEGY from the 70's or 80's. It's also an advantage that all program packages can, somehow, import and export the files.

I agree with David, that the headers doesn't need to be in the same file as the data. I'd prefer to have the trace headers readable as ASCII files whiles the data can be binary, but preferable standardized or at least defined in the header. ASCII trace headers are also easy to import for GIS applications, SQL or spreadsheets.

The rsf format (developed from SEPlib), could have been exactly what I was looking for, but trace headers are not written to the header file, but instead placed in separate files. This is not bad for the processing workflow, but it makes it difficult to export files.

The perfect solution, from my limited experience, would be something like a rsf file, with file header and trace headers in the same .rsf. I've been suggesting it to some fellow Madagascar users, and the argument against it is that in large projects the header file would be very (too) large. However, I don't see that as a problem, rather another argument to have the trace headers in an easy searchable and as far as I know there are no limitations of ascii files size.

Tbb


## ANSWER

One thing that already exists is the ph5 format from the PASSCAL group: https://www.passcal.nmt.edu/content/ph5-what-it

It is essentially a port of the SEG Y format that removes some restrictions and adds a bunch of new meta information. It is based on HDF5 and is currently mainly intended as an archival format although I don't see why it could not also be used as a processing format.

Disclaimer for the following paragraphs: I am one of the guys behind it. We have recently been developing a new data format including data provenance for seismology also based on HDF5. Currently it is mainly suitable for passive source data, e.g. earthquakes and stations recording the ambient wavefield - our community has very extensive and well established standards for the meta information which we just incorporate. It would be some work but the concept of seismic sources and receivers translates to the active source case so that might be a valuable direction to investigate. More information: http://seismic-data.org

Lion Krischer


## ANSWER

The SEG has a sitting committee for SEG-Y revision 2, which has debated the need to adopt changes to SEG-Y since the last revision that are needed (such as removing trace length limits for continuous data, formats for passive data, etc.). If you want to do something about what SEG-Y for the future should look like, You should join the SEG and volunteer for the committee.

People need to remember that SEG-Y is an interchange/exchange format, not a processing format. Acquisition formats (like SEG-D) are good for field acquisition. SEG-Y is for data exchange. In fact, is was a replacement for the older SEG Exchange format (SEG-X, get it?). No one will wade into the argument about a standard for a processing format because it has even more issues of compatibility and legacy than SEG-Y.

txpaulm
