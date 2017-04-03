# CONVERSATION ABOUT SCANNING SEGY on Software Underground

> from the #general channel.
> If anyone captures convo after 3 April 2017, please dump to a new file.

----- March 14th -----

nthorder [12:55 PM] 
does anyone know of libraries, preferably with Python interfaces, that allow efficient parallel reading/writing of segy shot gathers? I have looked at obspy, the 60 North one, and Statoil's recent segyio, but none of those are quite what I am looking for.

matt [1:06 PM] 
Thanks for pointing at `segyio` — I was not aware of that.
@nthorder So the problem is they don't do parallel natively?
Parallelizing i/o sounds really... hard. (edited)

nthorder [1:09 PM] 
there are a few problems. One is that most or all of them want to scan the whole file when you open it, which is not a good idea if opening it on many nodes simultaneously. The second is that none of them are really designed for reading shot gathers. The only way it works really is if your segy is already sorted into shot gathers, and you have the same number of traces in each gather. Then you can manually request those blocks of traces. I was hoping for something a bit more tailored to the (presumably fairly common) case of wanting to read shot gathers.
what I need, I think, is a tool that reads the segy and builds a database containing the trace ids for each shot. Then when running on many nodes, they just need to read this database to find out where in the segy they need to read.

matt [1:16 PM] 
ObsPy can read only the headers, then you can query the file arbitrarily. That's what I'd do.
 ```from obspy.io.segy.segy import _read_segy
section = _read_segy(filename, headonly=True)
```
(edited)
If the headers contain good indices, you're set.

nthorder [1:18 PM] 
@matt ah apologies, that's what I meant when I said "scan the file". If I have tens of thousands of traces, even reading all of the headers with every node is going to be horrible
and if I have millions of traces, you can imagine...

matt [1:19 PM] 
Oh, I see what you mean... The problem with doing anything else is that I just wouldn't trust anything except the patterns in the trace headers.

joferkington [1:20 PM] 
On a semi related note, dask and segy should play somewhat well together in principle, though probably not as well for shot gathers. At any rate, it would be interesting to play around with. (Dask won't read segy natively, it would just be interesting to write a segy interface for it)

nthorder [1:22 PM] 
@joferkington Dask is exactly the tool I was planning to use, but I think I will still need a proper solution for scanning the headers in a parallel fashion and building a database of the trace ids needed for each shot gather
if I want to handle full modern datasets

joferkington [1:23 PM] 
Yeah, the problem with shot gathers is that you need to scan everything (ish) to know the relationship between gathers. (Like you just said) 
It's easier in the post stack world

nthorder [1:24 PM] 
I know of companies who have tools to do it, but I was hoping I wouldn't have to write an open source one myself as it's not an easy thing to do efficiently, especially when dealing with segy

lmoss [1:25 PM] 
Not familiar, but what does "scanning" entail?
I'm happy to take a link to read up on it myself ^_^

nthorder [1:25 PM] 
the other option I am considering at the moment is just using one of the existing segy readers to write a tool that converts segy into javaseis, which is a bit easier to work with.
@lmoss I think they just go through the segy and read each header. The tools I looked at were generally only designed for poststack segys, and so they were just looking for unique inline and crossline numbers, I think

lmoss [1:27 PM] 
Hmm, seems reasonable then to use dask and scan everything once, and hold only the headers in memory.

nthorder [1:32 PM] 
I admit to being quite shocked that there doesn't seem to be a good open source solution for this already. I am looking at open source projects that do RTM/FWI to see how they read in the segys, but so far they either just convert the segys into another format first (Madagascar), or only accept velocity models and then generate their own data.

nthorder [1:38 PM] 
@lmoss I don't think that can be easily achieved with any of the existing tools, as they handle the scanning themselves when you open the file, so it would require modifying the source code of the existing readers. If you want to keep the code simple, you would also need to make assumptions about each trace being the same length, etc., so that you could just starting reading the file on each node with a byte offset that corresponds to the beginning of a trace.

matt [1:41 PM] 
(Wondering if we can get @abingham to chime in... maybe 60 North is cooking something up along these lines?)

nthorder [2:18 PM] 
the very responsive developers of Statoil's segyio have added an option for me that prevents it from scanning the trace headers when opening the file. This will at least allow me to manually get something going.

matt [2:19 PM] 
I notice they are responding to issues as I install the library...

alessandro-adm [4:46 PM] 
`segyio` from Statoil?
and no links!
here it is: https://github.com/Statoil/SegyIO 
interesting, if somebody is using it can he please tell the rest of us what are the differences with obspy? assuming one uses these libraries just to read/write segy...
also it would be good to have a conda package so I could try it at work too
from what I see on the readme, it's supposed to be built from source (something I can do on my mac but not on my work pc)

matt [5:38 PM] 
@alessandro-adm I just gave it a try... https://github.com/agile-geoscience/geocomputing/blob/master/notebooks/Read_SEG-Y_with_SegyIO.ipynb 
It seems to be aimed at handling nicely formatted 3Ds, so it's a bit more fiddly on 'normal' data. And I can't figure out how to read headers.
(for comparison, here's how I do ObsPy, possibly non-optimal, https://github.com/agile-geoscience/geocomputing/blob/master/notebooks/Read_SEGY_with_ObsPy.ipynb )

hassan_s [10:41 PM] 
@matt seg-y header as in EBCDIC header ?


----- March 15th -----
matt [1:01 AM] 
@hassan_s Any header... File binary header. File ASCII/ebcdic header. Trace headers.

sebastiang [1:08 AM] 
you just can’t do work with SEGY files natively if you want to use the traces out of order. 
the work to scan the headers is equivalent to reading the whole file. disk reads in blocks (e.g. 4k at a time) such that reading 100 bytes every 4k bytes is no faster than reading the whole 4k bytes.
you always end up converting to something that puts the headers in a smaller index (as suggested here) and tiles the traces somehow.
exception worth noting: 3D post-stack surveys already filled in with blank traces, i.e. where you can directly calculate the offset of trace (inline,crossline) by knowing the survey. but that’s still inefficient. to generate, e.g. an elevation map, you have to scan the whole file.
you run into a similar problem (though on a different order of magnitude) even if you have the whole SEGY file in memory. Your CPU reads memory in blocks as well. Scanning trace headers _is_ faster in memory than reading the whole trace. But not by as much as you may have hoped.

matt [1:13 AM] 
SEG-Y rev3 will solve this problem. Discuss.

sebastiang [1:47 AM] 
Esperanto will put translators out of business. Discuss.

ahartikainen [3:26 AM] 
hey, what is an optimal way to save seismic data? Everybody are using seg-y but could there be a more optimal way? compressed hfd5 with problem specific structure?

bertb [3:53 AM] 
Nice broad question. SEG-Y is the industry standard but it is also the source of huge time loss of nearly anyone involved. The technology is way outdated; the concept is from the 1970s when there were tapes to store data on. There are a few alternatives but none has gained wide acceptance. If you don;t care about exchanging with others, stay clear of SEG-Y. If it's very important for you, you cannot cicumvent SEG-Y. Beware that SEG-Y is a badly defined standard; the SEG-Y revision 1 update has at least changed this by establishing the bare essentials like inline/crossline numbers. For OpendTect I have wasted at least a year of my life supporting this awful format.

mtb-za [4:26 AM] 
So what would you do as a format if you could start from scratch?
I mean, if you were given the latitude to develop a new one, what would it have, and how would it work? (edited)

bertb [Mar 15]
mtb-za: Well, for starters, I would not make a format but only an interface to access the data, together with implementations in all common languages. 
The underlying format would incorporate all goodies that we now expect from modern data access: bricking, compression, planned geometries for access, ...

jarmokivekas [5:02 AM] 
I'll admit I'm ignorant about SEG-Y, but I'd like point out google protobuf as cool tool for serializing arbitrary data in a fast and compact way if existing file formats don't work for you. Think JSON or XML, but as a binary format and with definition files that can be used for generating parsers and producers in different programming languages. https://developers.google.com/protocol-buffers/
Among other things, the openstreetmap planet files (the complete openstreetmap dataset) are distributed using a protobuf format these days

wouterk [6:14 AM] 
remember spooky things happening in segy if nr of samples gets beyond 2^31 (edited)

lmoss [6:27 AM] 
@wouterk or maybe 2^32-1 ? :smile: (edited)

wouterk [6:33 AM] 
yeah :slightly_smiling_face:

wouterk [6:42 AM] 
Apparently, rules for order of operation get flexible when I have a number in my head for too long:nerd_face:

ahartikainen [7:57 AM] 
should there be an open source flexible fileformat for seismics #increaseEntropy

sebastiang [Mar 15] 
ahartikainen: if there would be, it should be HDF5

ahartikainen [Mar 15] 
HDF5, msgpack (or any other binary json), Apache Arrow (feather), blosc. Which one is the / most modern. Could one use a panel of  dataframes as a datastructure with metadata?

sebastiang [Mar 16] 
blosc can be used internally by HDF5. Arrow would be handy for header information, since a column-store is almost perfect for that data. For very large seismic data, you need good tiling support and compression, which I think either points to HDF5, or to a blob-store-like solution (e.g. S3)

lmoss [8:41 AM] 
Would be a cool community project to setup no?
I always wanted to get into dask. Now that I'm working with large datasets (out of memory) I see myself getting into the need to try it out.

matt [9:20 AM] 
I would think Statoil might even like to be involved. So perhaps keep it close to (or in?) SegyIO?

bertb [Mar 15] 
matt: Just defining a format will not cut it. You need all sorts of things around it. I made a small overview of what I thought would be needed some time ago,  but I didn't really get response from potential sponsors.

bertb [Mar 15] 
I guess one of the things you need to worry about is that there are already candidates for such a format, like JavaSeis

matt [Mar 16] 
That requirements/design document looks like a nice piece of work, Bert. I will read it thoroughly.

alessandro-adm [11:08 AM] 
thanks @matt! I will check how the loading times are and I'll use whoever is faster. Or probably I'll still use obspy until I see a conda package around for segyio. Anyway obspy allows me to load 10Gb segys in 5-7 minutes, then I convert them to an xarray and all the other stuff I do is pretty easy, so from a practical standpoint I don't see any reason to switch (and yes, I do use well-formatted segy usually).
matt
(for comparison, here's how I do ObsPy, possibly non-optimal, https://github.com/agile-geoscience/geocomputing/blob/master/notebooks/Read_SEGY_with_ObsPy.ipynb )

leouieda [4:32 PM] 
uploaded and commented on this image: https://xkcd.com/927/
@ahartikainen don't need another standard, netCDF is already an amazing format with a bunch of readers in many languages, including xarray in Python witch integrates with dask.

leouieda [4:37 PM] 
There is also the ASDF (https://seismic-data.org/) by @krischer which I think is based on HDF5 (also the basis for netCDF).

ahartikainen [4:47 PM] 
That was the comic I was thinking when posting the suggestion.

mtb-za [4:49 PM] 
I was also thinking of that, but was wondering what might make seismic so special that it needs something that does not exist already.

matt [6:45 PM] 
Apropos seismic formats, there's some good discussion in the comments of this post... https://agilescientific.com/blog/2014/3/26/what-is-seg-y.html

matt [6:53 PM] 
Also this SE question... http://earthscience.stackexchange.com/questions/694/what-would-a-replacement-for-segy-look-like

dopplershift [7:25 PM] 
I’d be surprised if netCDF4 or HDF5 were insufficient for seismic data. But I’m biased in favor of netCDF.

joferkington [7:27 PM] 
HDF5 is pretty much ideal for seismic data. It's actually used internally by some interpretation packages. The only downside is getting people to agree on conventions for metadata and attributes.

dopplershift [7:28 PM] 
That’s a whole other problem (see http://cfconventions.org for some netCDF weather data conventions)
cfconventions.org
CF Conventions Home Page
Climate and Forecast Metadata also know as the CF Convention or CF Metadata

thomas [7:28 PM] 
both netCDF and HDF5 are used for marine CSEM data successfully, but I know CSEM data is a lot smaller

bertb [10:21 PM] 
I think there is so much more to the problem than just finding an 'optimal' format ...

[10:22]  
I made a document about this, it was some time ago, but I put it on my google drive:
https://drive.google.com/drive/folders/0B9Zs3IHsKfE0TG9rdFJkOFhlYkU?usp=sharing
there is an index.html in there
Hmm I find it hard to view stuff outside my own google drive. But readin the index.html will give you teh picture 
Sorry it is something I created with vimwiki, it's a directory with HTML files. Not sure how to share it properly. I have a zip file if anyone is interested.

> nb this file is in this repo at [../SeismicAccess/index.html](../SeismicAccess/index.html)

hassan_s [10:31 PM] 
seg-y is originally read from tapes sequentially, so it can be understood why it is not optimized for random access

bertb [10:31 PM] 
yes we can explain it, but do we all still need to use it? (edited)

hassan_s [10:31 PM] 
seg-y is still necessary. but the use case for processing/analytics is a different story 
I would propose a split up format with a header file and trace files

bertb [10:33 PM] 
Did you take a look at my documents?  
The big issues are on a data management level
it is not that hard to figure out some fats stuff that will work in some situations

hassan_s [10:34 PM] 
just read the updated convo while on the go 
not yet

bertb [10:34 PM] 
it will be hard to make something that will be usable and acceptable for everyone
not in the least because there are already contenders for the title of 'the new format', and they fight each other to death
what we'd need is a community project that focuses on the entire problem, not just on data formats

hassan_s [10:37 PM] 
just v quickly, grouping the main  header, index and linkage to trace locations in one file, then splitting up the traces into other files should be generalized enough for most use cases.

bertb [10:37 PM] 
I doubt that
For one thing, then there is no bricking for quick random access like time slices
compression is also interesting for cloud access
tehre is the issue of position keys
per-position aux data issues

hassan_s [10:41 PM] 
page swapping work the same way
index first, then jump to actual location

bertb [10:42 PM] 
try reading a single sample per trace for a terabyte cube  
ok I'll leave it at that, hope you can find a way to read the document

davidmholmes [11:53 AM] 
I went along to the SEG standards meeting at New Orleans, where this update was basically pushed through. Eight people turned up, one was there to present a certificate, one person was in the wrong room but was too embarrassed to admit it, and another didn’t say a word. So the bottom line is that the whole thing is really driven by three people. It blows my mind that one of the most complex and long living technical data standards has such little input from the broader community. It’s also the reason that there is no appetite to do anything other than spin the standard to plaster the cracks rather than re-imagining what a new standard might look like and how we might solve the broader set of issues with sharing and executing computation on seismic data in a very different universe from the one in which the original standard was conceived.
stateOfMind.Ranting(off)

sebastiang [11:56 AM] 
but in all seriousness, what is the value of multiple headers per trace?

ahartikainen [12:11 PM] 
So we should and could design better opensource format (under xarray + netCDF4)? I mean we have more than 3-4 people?

davidmholmes [12:35 PM] 
@sebastiang - a lot of people were complaining that the headers lacked the flexibility to incorporate extended metadata. It also made it difficult to make header configurations mandatory. This is why there are an almost infinite number of header configurations. By having multiple headers it means that there can be much more discipline with the primary header configuration and then additional headers can be used to store whatever additional weirdness you might want to capture.

sebastiang [12:36 PM] 
ah

davidmholmes [12:36 PM] 
If I have one more conversation with someone who says something like “oh yes, that’s a transitional survey so we relocate the X/Y locations so that we can store elevation and bathymetry there instead” I might just completely lose it....

[12:39]  
Of course if you’re really lucky and you’re shooting 3D on a surface with linear gradient changes, the elevation/bathymetry looks a lot like co-ordinates. All you need to do then is find a CRS that you can translate to vaguely the right area and you’re off to the races....

ahartikainen [12:53 PM] 
Surely I don't mean that we should replace segy. I just like the exercise. And how to optimize reading (e.g. reading just a subset etc)

matt [1:21 PM] 
Relevant re-post from a few days ago... http://seismic-data.org/
(for global seismology / earthquake records, but I like the approach and execution. This is from the ObsPy folks, @krischer et al)

ahartikainen [1:40 PM] 
That is great, I did not notice that before. Just wonder why they use astropy for times etc :thinking_face:

matt [3:46 PM] 
@nthorder This talk seems relevant to your question about parallel access to SEG-Y headers the other day https://www.youtube.com/watch?v=Y00Js6uPWI0&list=PLcsG4X8Zn_UDtkbuLj8K_gnjwytjK7BdW&index=10 cc @joferkington @sebastiang @hassan_s @bertb (one of the talks from the recent Rice HPC meeting that @davidmholmes posted about in #random)

davidmholmes [3:47 PM] 
Agreed. I just watched it. 
Our Irish CTO is a buddy of mine and he does a bunch of work with ICHEC. I've asked if he can make an introduction. I'd like to see if we can get Cathal to join Swung.
I presume it's not being open sourced until September so that DDN can get a head start on the competition.
The burst buffer idea is something we've played with but scaling it is not straightforward. They got great results with relatively small datasets but it's not obvious what would happen as the scale increased. 
Also, they're only supporting SEG-Y at the moment which is unusual given that the primary application is for processing.

bertb [7:44 PM] 
Everything gets different when you talk cloud access. You have to prepare for massively parallel access to data in buckets. This is on top of the stuff I wrote in the document I made earlier. All in all the hunt for a 'new format' is not going to be decided on the 'file format' level, but on 'interface access level'.
In terms of interface access, it is important to consider the various needs that the different groups have: acquisition, processing, interpretation. Different needs, different data. Header keys, access strategies, ... 
It would be so useful to have a common interface used all the way from acquisition (shallow, deep) to interpretation for 2D, 3D, 4,5,6, ... post/pre stack in different cloud types (Amazon, Azure, Google, ...) - and all that through interfaces that are as simple as possible ... A real challenge.


----- March 26th -----
fredrocks [10:50 AM] 
Interesting new repository to add to the netcdf, segy discussions : https://github.com/ar4/netcdf_segy


----- March 27th -----
sanandak [3:55 PM] 
@sebastiang - agreed that for millions of traces we need another solution. I suspect HDF5 is the way to go - they are pretty persuasive that the bottleneck is in the disk access, so their pipeline of compressing data before writing has big benefits.  Plus, hdf files have metadata that lets one access a piece efficiently.  Visualizing this large dataset is the next problem - and the mapping community may be the place to look, with tiles of differing resolution.  However, my experience with HDF has been problematic - the C and Fortran APIs are incredibly complex, and very easy to leak memory.  The only other language that I know of that works well with it is python, which unfortunately loses some of the speed benefits…  Note that HDF themselves have made a half-hearted attempt at a client-server setup, but their solution is json strings to pass data - which would be difficult for large datasets.

My project is, as you note, for “medium” datasets - for truly small datasets, one can pick with anything from matlab to (heaven forbid, exc*l - I can’t even bring myself to type it - but I know some who have used it!).  It is that region of 100s to 1000s of traces that we deal with here, and I think many academic settings do...

sanandak [4:19 PM] 
Apropos of nothing - for messaging between programs (and between machines), there is nothing better than ZeroMQ: http://zeromq.org.  It takes away all the heavy lifting of tcp sockets and lets you send messages in half a dozen lines of code… If you ever find yourself opening up the man page on sockaddr_in or whatever, stop!


----- April 1st ----- (but not joking)

In an almost certainly misguided attempt to get my arms around things, I made a repo with a lot of stuff about 'the next gen seismic data format'. https://github.com/softwareunderground/xsdf 
Heads up @bertb - your HTML doc is in there, please let me know if you wish it wasn't.
cc @nthorder @sebastiang @mtb-za @lmoss and everyone else who was interested. I have no idea where this needs to go. Plan A is probably to figure out if ASDF is adaptable enough to be the thing. http://seismic-data.org/

dopplershift [5:55 PM] 
@matt Is netCDF excluded?

bertb [6:48 PM] 
@matt: Nice! I had already given up.  A bit of a pity it shows the HTML code rather than the page when you click on it ...

josephwinston [7:41 PM] 
The requirement not to have any external dependencies is too harsh and I don't support this idea.  Minimal dependencies, is something I'd like to see.

bertb [8:01 PM] 
Agree. Minimal dependencies, and make sure the dependencies are on a project that is very well alive and kicking. And of course no proprietary stuff ...


----- April 2nd, 2017 -----
leouieda [12:01 AM] 
Yep. Without hdf5 you might loose access to cool things like http://www.opendap.org/ (there is a Python client http://pydap.readthedocs.io) and all the maths infrastructure like xarray which parallelizes a lot of things through dask. 
I like the idea of "Language and platform agnostic.", though that usually means writing a C library.

sebastiang [1:47 PM] 
I think the general problem with a standard seismic format is that there are several standard ways you want to ~process~ access seismic. transporting as a tape, reading off of geophones, random access for interpretation, access for processing, all suggest different tradeoffs. (edited)
SEG-Y is bad, but history has proven it’s good enough.

ahartikainen [5:03 PM] 
Transporting as a tape? Who still uses tape? :scream:

jonnyford [5:43 PM] 
Just from my experience in a processing shop - a surprisingly large number of (major) companies. Granted, these days most only insist on getting a tape copy as part of their archiving/backups process but there are a good number that still use tape as their primary format to move data around. Things are changing but tape is still clinging on as "common denominator" medium for seismic interchange in the oil and gas industry.

bertb [7:08 PM] 
And we are working like mad to support direct cloud access. Putting everything in the cloud is the new thing, flexible computing power, flexible disk space, cost savings. The challenge is not to go through any file system but access the buckets/blobs directly for huge speedup and cost saving. Will there be tapes in 5 years? Does anyone still work with punched cards or floppy disks?

quicksilversystems [11:53 PM] 
joined #general


----- April 3rd, 2017 -----
matt [8:37 AM] 
@bertb I've seen a few people making virtual file systems for their cloud storage, so other apps are effectively supported 'for free' (as if)... have you come across that?

hassan_s [9:39 AM] 
@bertb I have not been providing input on seismic format proposal because I felt it'd be irresponsible to comment but not actually work on it. Time is precious however but I still would like to contribute, so here is my 2 cents. Something like protobuf or hdf5 is sufficient, and you will need several specs based on each use case. I'd rather have each software read seg-y then produce a standard cache format based on the use-case.  
with SSD, reading should be fast enough that we don't have to  super-optimize the format to get as much sequential read pattern as possible. 
It's cheaper to buy more SSD compared to the time spent trying to optimize for all use cases and getting others to adopt them.

bertb [11:05 AM] 
I think you are not right; Fast storage is just part of the problem. Data over networks. Data in cloud storage accessed by virtual machines residing in the cloud. And more. There will never be a shortage of use cases where performance is an issue. Whether HDF can solve everything? I would not know. All I know is that everything that was tried before failed - because the format was plainly rejected because of these concerns. Just waving them away is IMO not a good attitude.

sanandak [1:06 PM] 
Nice post on chunking in HDF5: http://geology.beer/2015/02/10/hdf-for-large-arrays/

geo_leeman [3:33 PM] 
To echo @dopplershift - is there something ruling out netCDF? Uses HDF5 with an easy API.

sebastiang [3:51 PM] 
If I ran this slack I’d suggest a dedicated channel at this point. #death-to-segy or something. :smile:

matt [4:35 PM] 
Your wish is my command. #xsdf At least we will get #general back :slightly_smiling_face:

> At this point the convo moves to #xsdf I hope and it's probably a good time to start a new archive.
