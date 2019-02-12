# Asteroid Detector
A buddy module to Photometry Pipeline for asteroid detection and reporting.

<img src="http://www.rankinstudio.com/dnloads/AsteroidDetector2.JPG" width="800px" />

Video of Asteroid Detector in Action: https://www.youtube.com/watch?v=ayBl_3fZ760

# Introduction

This is a guide on how you can get Asteroid Detector up and running. Just a fair warning, this will be rather long and complicated. If you are not familiar with Linux or Python, it may prove to be difficult to get working. It is a personal project I built for my own needs and am offering free without support. If you find any massive bugs or errors in the guide please let me know. You need an MPC observatory code to submit observations of asteroids. Also feel free to contribute if you know Python!

Asteroid detector works by using Photometerypipeline (PP) to extract sources from your images, and calibrate them. It works by stacking groups of images with Siril to get better S/N ratios then looking in the stacked images for asteroids, it also works with single images. These sources are stored in .db files. Asteroid Detector scans these .db files for moving objects based on some search parameters you provide. If found, it uses DS9 to plot these asteroids for you to confirm if they are real or not with a conformation window. It will completely build your MPC report files. It can also parse the MPC to identify asteroids, and has the MPC's object rating tool (Digest2) built in so you can see the orbit probabilities as you scan through the detections. Asteroid Detector and PP create a lot of files during calibration and object searching. You can clean most of the unused files up from the GUI. The MPC report file (Report.txt) is saved in whatever directory you are working in. Asteroid Detector assumes you have somewhat accurate RA / DEC information in your fits headers for plate solving.

# Setup
A Linux installation, I'm running Ubuntu 16.04 in VirtualBox
I setup a shared folder between Windows and VirtualBox to move files into Linux

Install Source Extractor, SWarp, and Scamp
```
sudo apt-get update
sudo apt-get install sextractor
sudo apt-get install scamp
sudo apt-get install swarp
```
I highly recommend reading over the documentation for Source Extractor, SWarp, and Scamp to get a basic idea of how they work together.

The next step is to get Photometrypipeline (PP) running: <a href="http://photometrypipeline.readthedocs.io/en/latest/install.html" target="_blank">PhotometeryPipeline</a>

There are detailed instructions at that site for getting PP up and going.
Install PP in /home/YourUsername/photometerypipeline/

To get SWarp running right from PP, I had to modify the pp_combine file like so:
```
In pp_combine line 234:
    commandline = (('swarp -combine Y -combine_type %s -delete_tmpfiles '+
change to:
    commandline = (('SWarp -combine Y -combine_type %s -delete_tmpfiles '+
```

From there, you'll have to make some changes to your bashrc file
Edit the bashrc from terminal with nano
```
nano ~/.bashrc

append these lines to the end, replace username with your username in Linux

# photometry pipeline setup
export PHOTPIPEDIR=/home/username/photometrypipeline/
export PATH=$PATH:/home/username/photometrypipeline/
export PATH=$PATH:/home/username/AsteroidDetector/digest2/
alias python=python3
```

This makes files in these folders available to run anywhere from the terminal window

To test this, you should be able to open a terminal and just run a command like pp_run or pp_combine. It should prompt with options you are missing.
```
david@david-VirtualBox:~$ pp_run
usage: pp_run [-h] [-prefix PREFIX] [-target TARGET] [-filter FILTER]
              [-fixed_aprad FIXED_APRAD]
              [-source_tolerance {none,low,medium,high}] [-solar]
              images [images ...]
pp_run: error: too few arguments
```
If you do not get this result, PP is not installed correctly, or not referenced correctly in the bashrc file.

PP requires specific configuration for your telescope equipment. The documentation for this is on the page above for PP. If you need help, here is an example of my "mytelescope.py" file:
This is made by copying the provided "telescopes.py" file and making modifications to match the information in your image fits headers. If you compare the differences between the file below and the provided "telescopes.py" file you wills see the changes I had to make to get the software working.

``` python
# MYTELESCOPE setup parameters
twelveinchccd_param = {
    'telescope_instrument': '12IN/CCD',  # telescope/instrument name 
    'telescope_keyword': '12IN_CCD',  # telescope/instrument keyword
    'observatory_code': 'V03',  # MPC observatory code
    'secpix': (0.65, 0.65),  # pixel size (arcsec) before binning

    # image orientation preferences
    'flipx': True,
    'flipy': False,
    'rotate': 0,

    # instrument-specific FITS header keywords
    'binning': ('XBINNING', 'YBINNING'),  # binning in x/y
    'extent': ('NAXIS1', 'NAXIS2'),  # N_pixels in x/y
    'ra': 'OBJCTRA',  # telescope pointing, RA
    'dec': 'OBJCTDEC',  # telescope pointin, Dec
    'radec_separator': ' ',  # RA/Dec hms separator, use 'XXX'
    # if already in degrees
    'date_keyword': 'DATE-OBS',  # obs date/time
    # keyword; use
    # 'date|time' if
    # separate
    'obsmidtime_jd': 'MJD-OBS',  # obs midtime jd keyword
    # (usually provided by
    # pp_prepare
    'object': 'OBJECT',  # object name keyword
    'filter': 'FILTER',  # filter keyword
    'filter_translations': {'Luminance': 'V'},
    #'filter_translations': {'Red': 'V'},
    # filtername translation dictionary
    'exptime': 'EXPTIME',  # exposure time keyword (s)
    'airmass': 'AIRMASS',  # airmass keyword

    # source extractor settings
    'source_minarea': 3,  # default sextractor source minimum N_pixels
    'source_snr': 2.5,  # default sextractor source snr for registration
    'aprad_default': 4,  # default aperture radius in px
    'aprad_range': [3, 10],  # [minimum, maximum] aperture radius (px)
    'sex-config-file': rootpath + '/setup/12IN_CCD.sex',    
    'mask_file': {},
    #                        mask files as a function of x,y binning

    # scamp settings
    'scamp-config-file': rootpath + '/setup/12IN_CCD.scamp',    
    'reg_max_mag'          : 16,
    'reg_search_radius'    : 0.5, # deg
    'source_tolerance': 'high',

    # swarp settings
    'copy_keywords': ('TELESCOP,INSTRUME,FILTER,EXPTIME,OBJECT,' +
                      'DATE-OBS,TIME-OBS,OBJCTRA,OBJCTDEC,SECPIX,AIRMASS,' +
                      'TEL_KEYW,XBINNING,YBINNING,MIDTIMJD'),
    #                         keywords to be copied in image
    #                         combination using swarp
    'swarp-config-file': rootpath + '/setup/12IN_CCD.swarp',

    # default catalog settings
    'astrometry_catalogs': ['GAIA'],
    #'photometry_catalogs': ['SDSS-R9', 'APASS9', '2MASS']
     'photometry_catalogs': ['APASS9']
}

##### add telescope configurations to 'official' telescopes.py

implemented_telescopes.append('12IN_CCD')

### translate INSTRUME (or others, see _pp_conf.py) header keyword into
#   PP telescope keyword
# example: INSTRUME keyword in header is 'mytel'
instrument_identifiers['QHY CCD QHY163M-ea87d84'] = '12IN_CCD'

### translate telescope keyword into parameter set defined here
telescope_parameters['12IN_CCD'] = twelveinchccd_param
```

You'll also notice that the YOURCCD.swarp, YOURCCD.scamp, and YOURCCD.sex files are referenced as well. Copy them from another scope in the PP setup directory and rename them to match the paths you provide in the "mytelescope.py" file above. There is a lot of other information in here you have to fill out from your CCD header. Just read it over and you'll figure out what goes where.

I changed the photometery catalogs to only " 'photometry_catalogs': ['APASS9']" as I was having issues getting the others to work well. APASS9 seems to work good for calibrating magnitudes in your images. You can commend that line and uncomment the one above to use the other two catalogs. 

You'll now need to install DS9. <a href="http://ds9.si.edu/site/Download.html" target="_blank">DS9 Download Page</a>
This is another program you'll want to get familiar with. I extracted DS9 to /usr/local/bin/ so it could be ran from the command line anywhere. You can put it anywhere if you modify your bashrc file with the PATH like we did above. Make sure you can launch DS9 from a terminal window by simply typing DS9. 

There are a lot of python modules that Asteroid Detector relies on. Most of them come with Ubuntu 16.04 but some you'll have to install with apt-get or pip
Make sure all of these Python3 modules are installed: pyds9, pickle, glob, datetime, astropy, bs4, subprocess, shlex, tkinter, requests, random, string, sqlite3, numpy

Install Siril for stacking
http://free-astro.org/index.php/Siril:install

From here, you should do some extractions using pp_extract and change your .sex file to output a "check.fit" image of APERTURES so you can see how your extraction settings are working.

# Asteroid Detector
Asteroid Detector expects data in a certain directory structure:
```
Folder(Sequencename)
   (Folder1)
      -Image1
      -Image2
      -Image3
      -Image4
   (Folder2)
      -Image1
      -Image2
      ect...
   (Folder3)
   (Folder4)
```
It stacks 4 groups of images to achieve better S/N ratio for searching. With my setup, I typically shoot 35 X 40 sec subs, then group them in stacks of 5. So images 1-5, 11-15, 21-25, 31-35. 

After you verify that all the above is working, getting Asteroid Detector installed is easy. Just download this archive and extract to /home/YourUsername/AsteroidDetector

In the Asteroid Detector directory you'll see a desigused.txt file. This keeps track of the designations you send to the MPC so it doesn't create duplicates. 

 # The Settings File

There is also an obsconfig.txt file. This contains the settings for Asteroid Detector:

Here is my settings file
```
### Asteroid Detector Settings ###
Observatory Code: V03
Name: David Rankin
Observer: D. Rankin
Measurer: D. Rankin
Telescope: .30-m F4 Reflector + CCD
Email: David@rankinstudio.com
Initials(2): DR
PP Path: /home/david/photometrypipeline/
Begin Exposure Key: DATE-OBS
Exposure Time Key: EXPTIME

### Defaults ###
Default FWHM Minimum: 1.5
Default Asteroid Search Radius: 30
Default Star Search Radius: 3
Default Limiting Mag: 20.5
Max Resid: 0.45
Slop Factor: 4
Star Lim Mag: 21
Default File Open Dir: /home/david/Desktop

### DS9 Settings ###
DS9 Text Color: green
Invert images?: no
Image Keyword: Image
Blink Inverval: 0.2
DS9 Font Size: 15

### EXTRACTION SETTINGS ###
PP Register Command: pp_register -snr 6 -minarea 4.5

# FINAL EXTRACTION COMMAND
PP Photometry Command: pp_photometry -snr 1.8 -minarea 2.4 -aprad 4

# FINAL EXTRACTION COMMAND
PP Calibrate Command: pp_calibrate -maxflag 5
```

The settings are pretty self explanatory. Reference DS9 manual for options with color and blinking interval ect.

Search parameters under Defaults are all specified in arc seconds. 

"Image Keyword" is very important. This is the keyword in the file names of your images. This is how AsteroidDetector finds your files. Please use ".fits" extension.

"FWHM" ignores objects in the database with an FWHM in arcsec smaller than what you specify. This is handy so you are not subtracting noise as stars.

"Default Star Search Radius" and "Default Asteroid Search Radius" are self explanatory.

"Default Limiting Mag" only considers sources brighter this mag for transient objects (asteroids)

"Max Resid" is the maximum acceptible residual you will accept (deviation from a line) for the object to be considered an asteroid.

"Slop Factor" while searching, vectors are built around the first two detections based on the time elapsed between images. As you search for a 3rd hit, this is the factor of slop in arc seconds you will search in to find an object near the predicted location for the 3rd hit. This is also used for the 4th hit with a scale factor of 1.5 times the original slop factor. This effectively creates a cone that increases the further you get from the first two source hits.

"Star Lim Mag" prevents the pipeline from subtracting asteroids that are near noise, where the noise may accidently be considered a star. It ignores any sources fainter than what you specify here.

"PP Register Command" coveres the initial registration commands for PP. This is not the final extraction command, it is only used to calibrate the images. So you want to specify brighter soures here. 

To make a quicklaunch for it, create a new file on your Desktop called "AsteroidDetector.sh" or whatever you'd like. In this file put:
```
gnome-terminal -e "python3 /home/YourUsername/AsteroidDetector/Main.py"
```
Right click on the file, go to properties, and make sure it is set to executable under permissions. 

# Using Asteroid Detector

From here, you just launch it and select your working directory with the structure specified above. Click on "PROCESS IMGS". Be patient, this can take a while

Once this completes and you have good extractions in the .db files, you don't need to run it again. You can change your asteroid searching parameters and run any number of searches on the .db files you'd like without having to re-register the images.

Once asteroids are extracted, you'll notice the list window fills in with objects. These can be navigated with the up and down arrows on your keyboard. To "check" if an object is in the MPC database and get its probabilities without writing it to the final report, press "c" when the object is hilighted. If you are content it is real and wish to write it to the report, press "enter/return" on the higlihgted object. 

Exmaple output from pressing "c" on a suspected asteroid:

```
Asteroid  7 / 19

###### RESIDUALS ######
-0.00
+0.01
-0.03
+0.01

 0.01  Asteroid mean residual

Searching MPC, please wait..

Information for TMP001
##########################################
#####    MPC SEARCH RESULTS BELOW    #####
##########################################

      TMP001  24215                    07 01 31.5 +19 55 09  17.8   0.0E   0.0S    39-     4+   17o  None needed at this time.
MPC desig:  24215

############################################
#####    OBJECT PROBABILITIES BELOW    #####
############################################
Desig.    RMS Int NEO N22 N18 Other Possibilities
 24215   0.05   6   6   2   0 (MC 1) (MB1 57) (MB2 20) (MB3 12) (JFC <1)

 Finished searching and scoring object. 

```

# Advanced Configuration / Testing

I have included tools to troubleshoot extraction:

"Show Trans" displays all extracted transient objects.
"Show Extraction" with image number shows the extracted objects for the image specified.

These are helpful with tweaking your extraction commands in the settings file.

Once this is all working, you need to verify you are getting good extraction from source extractor, and that your residuals are good across the frame. My images are warped pretty bad by the optics because my FOV is pretty large. I had to change the setting in my PP/setup/MYSCOPE.scamp file: DISTORT_DEGREES to 3 in order to account for this. Getting somewhat familiar with Source Extractor, Scamp, and SWarp will help you a lot.

I also changed DEBLEND_MINCONT to 0.0001 in my 12IN_CCD.sex file to get sources that were close together separated. 

You can access the two most common Source Extractor settings in the "mytelescope.py" config file above. You can see what I chose for mine.

This has been a lot of fun working on and in the long run will save me A LOT of time loading images, solving them, stacking them, and then searching for asteroids. If you can get it running I hope you also find it useful.

# Issues With PP

I had to modify a few things in photometry pipeline to fix some minor issues. One was with the skycenter function, another was with how it reads the coordinates in the image headers. I have included my photometrypipeline directory in "Extras" minus the git and example files so you can compare files if you run into issues. 

I ran into a few issues with Photometry Pipeline. Here is what I did to work around them:

There were some leading blank spaces on my RA / DEC coordinates causing problems with pp_prepare. I modified it like so:
# pp_prepare.py:

```
        if obsparam['radec_separator'] == 'XXX':
            ra_deg = float(header[obsparam['ra']])
            dec_deg = float(header[obsparam['dec']])
        else:
            ra_string = header[obsparam['ra']].split(
                obsparam['radec_separator'])
            #print(ra_string)
            if (ra_string[0]==''): <-- Added this
                ra_string.pop(0)
            dec_string = header[obsparam['dec']].split(
                obsparam['radec_separator'])
            if (dec_string[0]==''): <-- Added this
                dec_string.pop(0)
```

I was also having issues with the skycenter function in pp_calibrate which was causing the image calibration to fail as it was not parsing the viezer servers correctly:

# pp_calibrate.py:

Comment this line out:
``` python
#ra_deg, dec_deg, rad_deg = skycenter(catalogs)
```
Replaceed it with:
``` python
#------------------- HACK ---------------------------------------------------_#
    #NOT RETURNING RIGHT POSITIONS, OVERWRITE
    hdulistT = fits.open(filenames[0], ignore_missing_end=True)

    header = hdulistT[0].header
    ra_new = header[obsparam['ra']].strip()
    dec_new = header[obsparam['dec']].strip()

    c = SkyCoord(ra_new, dec_new, frame='icrs', unit='deg')
    print(c.dec.deg, " From PP_Calibrate Hack")
    print(c.ra.deg * 15, " From PP_Calibrate Hack")

    ra_deg = c.ra.deg * 15 #Convert to hours
    dec_deg = c.dec.deg
    rad_deg = 1.0 # Manually set the area to be retrieved. 
# ------------------- HACK ---------------------------------------------------_#

```




