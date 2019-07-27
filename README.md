

## Tue 16 Jul 2019 12:32:42 AM CEST
Decided to go with HMI HARPs because they are extracted patches of active region; They are standard HMI vectorgram but they extracted patches where they found AR;
There is kind of 1:1 NOAA AR to HARPNUM mapping, but some AR can be in multiple NOAA AR; Got full list of AR -> HARPNUM;

TO DO: Find all HARPNUMP for all AR that Jan sent you so you can make JSOC query. (Think about python vs ssw for data retrival; Python has nice and clean API)

## Sat 20 Jul 2019 05:06:54 PM CEST

Created a query to map NOAA AR to HARPS list:
```
cat AR_LIST | grep -Ev "^#" | awk '{ print $1 }' | xargs -I '{}' grep '{}' HARPNUM_NOAA.txt  | awk '{ printf "%.4d\n", $1 }'
```

It will take first column from AR_LIST sent by Jan, and map it to HARPNUM_NOAA.txt file retrived from JSOC.
Note that some HARPS will have multiple NOAA AR regions in it, in our case that is one `(HARPS:284  NOAA:11133,11134)`

## Tue 23 Jul 2019 11:38:21 AM CEST

If you want to create query on HMI sharp/harp but you only know NOAA AR number and start date, you can write down query like this 

```hmi.sharp_720s_nrt[][2013.12.07/21d][? NOAA_ARS ~ "11923" ?]{**ALL**}```

- `[]` is HARPNUM; Empty because we dont know it
- `[2013.12.07/21d]` - Start date and duration
- `[? NOAA_ARS ~ "11923" ?]` - Query string to filter specific AR
- `{**ALL**}` - i think this is record limit, but im not sure.


Also, manual is here http://jsoc2.stanford.edu/ajax/RecordSetHelp.html (example 6)



## Tue 23 Jul 2019 12:17:36 PM CEST

Problem with query above is that we are using nrt (near real time data) its fast track pipeline for live checking of data (i assume), if we change to regular 720 sharp, query is working with HARPS number as expected.

You can see different sharps formats here http://jsoc.stanford.edu/doc/data/hmi/sharp/sharp.htm, i've opet out for trying hmi.sharp_720s (and later on hmi.sharps_cea_720s) and abandon nrt.

So SHARP query at the end should be like ```hmi.sharp_720s[3604]{**ALL**}```

Also, this fixes missing NOAA -> HARPNUM problem and removes query by date and NOAA_ARS number; Its cool

## Tue 23 Jul 2019 04:04:14 PM CEST

Downloaded SHARP data for HARPNUM 3481 and 3604 (corresponding to NOAA AR 11923 and 11950); We downloaded both SHARP 720s and SHARP CEA 720s; Currently on `chromosphere:/seismo4/lzivadinovic`

| HARP |  NOAA  | JSOC_ID           | CEA_JSOC_ID      |
| -----|--------|-------------------|------------------|
| 3604 |  11950 | JSOC_20190723_431 | JSOC_20190723_505| 
| 3481 |  11923 | JSOC_20190723_429 | JSOC_20190723_489|

CEA - are reprojected to local frame, ambiguity resolved magnetic data (Br, Bp, Bt) see sharp webpage for more details; They are also rotated for 180 degrees so that north is up, and east is right. TAKE CARE WHEN COMPARING TO REGULAR SHARP DATA!

This data is bassicaly prepared for analysis.

## Tue 23 Jul 2019 11:30:05 PM CEST

Tried this out https://github.com/cdiazbas/enhance looks fun and works well. Tested on sunspots with lightbridges, looks really good on enhanced images. Note that you get better results if you normalize data by quiet sun. In the end, i dont see any use of it because we are not interested in small scale structures evolution.

## Wed 24 Jul 2019 11:27:45 AM CEST

Created ipynotebook for creating mp4 files that represents data animation/evolution. 

Add requirements.txt (its mess, but at least its working with enhance)

#### Note on requirements provided 

Please use virtualenv if you dont want to mess your system!

You need to install specif version of packages, simple `pip install -r requirements.txt` will not work because there are packages that were compiled/installed via git

IIRC, these are onyl two i used (keras and tf contrib)

```bash
sudo apt install imagemagic imagemagick-common ghostscrip libtk-img libtk8.6 libcfitsio
pip install sunpy[all] #in zshell use pip install sunpy\[all\]
pip install astropy[all] # -||-

pip install git+https://www.github.com/keras-team/keras-contrib.git

#also you need to clone repo and run this https://github.com/keras-team/keras-contrib#install-keras_contrib-for-tensorflowkeras
#its some wraper thing i dont quite understand
```

I recommend going `pip install -r requirements.txt` untill you encounter error for some package, remove that package from txt file, and rinse and repeat with pip install unitl you get over all packages. Then run install `sunpy[all]` and `astropy[all]` and keras contrib with both pip and cloned repo (this conversion stuff) and that should be it



## Wed 24 Jul 2019 02:07:32 PM CEST

Tried PCA reduction for removing limb darkening, it only works with data that are near limb. (comment bellow pasted from animate.ipynb, see animate.html for prerendered notebook with data and comments)

Idea is that because there is no obvious intensity gradient, primary component of SVD is not actually limb darkening but some small scale variation across image. This leads to adding some artifacts that are not related to limb darkening, so this method is not suitable for removing limb darkening near disk center. Better idea is to create limb darkening function for this specific filter using some curve $f(r,\lambda)$, where $r$ is distance from disk center, and find $r$ for every pixel and reduce it that way.



## Wed 24 Jul 2019 04:35:32 PM CEST

Quick memo: DONT FORGET TO FIX HEADER ON IMAGES ONCE YOU RUN ENHANCE! When running enhance, your images are upscaled by factor of 2, so crval1/2 and crpix1/2 changes acordingly

## Wed 24 Jul 2019 05:55:17 PM CEST

Created function for limb darkening using http://articles.adsabs.harvard.edu/pdf/1977SoPh...51...25P (see limb_darkening.html/ipynb) for more details;

I cant figure out coordinate system mapping of SHARP CEA. Im having problem calculating distance from sun center in r_sun using CRVAL, near limb and center images have very similar values for this keyword. Not sure what is happening, but i found out that someone wrote conversion http://hmi.stanford.edu/hminuggets/?p=1428 (look at [S1] reference). I thought that it should be crpix = center pixel coordinate in pixels, crval = centar pixel coordinate in some coordinate system (Carrington in this case, but comparing near limb and center images shows different results). Need to read conversion to figure out what is happening. Also, http://jsoc.stanford.edu/doc/keywords/JSOC_Keywords_for_metadata.pdf crunit2 (y axis) for CEA should be sin latitude (or something like that) but in our data is degrees. IDK.


## Thu 25 Jul 2019 06:34:18 PM CEST

Finally figured how to easily and quicky transform images to helioprojective coordinates (distance from disk center in arcsecond) so we can now easily correct for limb darkening.

There are some concerns regarding general coordinate transformation for HMI data. Some people say that SDO does not report (in header) same satelite possiotion for HMI and AIA data taken at the same time, and also there is still discussion on how carrington system is deffined (if i understood correctly some guy in chatroom) (see https://github.com/sunpy/sunpy/issues/3057) 

Albert Y. Shih:
"Re HMI, do keep in mind that we don't have a good observer coordinate from the FITS headers (see  issue #3057), so conversions between HPC and HGC in SunPy may not match whatever is done elsewhere (and that's on top of the fact that people don't necessarily agree on the definition of Carrington longitude"

Big thanks to Stuart Mumford, lead developer of sunpy for helping me figure this out (https://github.com/Cadair).

General comments regarding code detalis and procedures were added to limb_darkening notebook. (for non interactive preview open limb_darkening.html)


##Thu 25 Jul 2019 06:43:29 PM CEST

Reading material (helpers):

https://arxiv.org/pdf/1309.2392.pdf

https://nbviewer.jupyter.org/github/mbobra/calculating-spaceweather-keywords/blob/master/feature_extraction.ipynb

http://jsoc.stanford.edu/relevant_papers/ThompsonWT_coordSys.pdf


## Fri 26 Jul 2019 02:59:25 PM CEST

Wrote wraper function that will do limb darkening clearing of images. Its bassicaly nicely written wrapper for stuff we did yesterday. Note that we did not change header of original images (for example mean intensity keyword is incorect, etc...) everything else should be fine.

I did not wrote comments in script, because limb_darkening.ipynb/html contains detailed instruction for one image reduction; Also, to convert ipynb to actuall python script that you can actually run with `python script.py` one can use `jupyter nbconvert --to script remove_limb_darkening_for_dataset.ipynb` it will create file with same name but with .py extension and removing all json embedded data.

## Fri 26 Jul 2019 09:32:17 PM CEST

Tried https://docs.sunpy.org/en/stable/api/sunpy.map.GenericMap.html#sunpy.map.GenericMap.resample and its working as expected. It's even updating headers, so that coordinate frame stays fixed and you can use it later on. It does not update naxis1/2 (dimension of image), but that is no big deal, coordinate delta per pixel, and reference center pixel is updated. Would recommend further usage of this procedure! (maybe submit PR to fix this issue?)

There is also small artifact, that shows up as 0 value padding on the outtermost pixels. So your whole image (rescaled to finner grid using cubic interpolation) is sorounded by 1px box with value 0, but we can just ignore those, normalized histograms looks identical.

Also, i've changed enhance codebase to allow writing back fits headers from enhanced images. I will need to transform header to perserve resampled data coordinate grid, my plan is to use code from sunpy.map.resample as base because they have it very clearly written how they transform header to perserve coordinate grid. But, for now, you insert image, you get back image with headers, original enhance removed headers from image and only saved data back.

## Sat 27 Jul 2019 07:29:24 PM CEST

Fixed enhance header data and write header, if PR doesn't get merged, we could use my fork. https://github.com/lzivadinovic/enhance
