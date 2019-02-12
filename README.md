# Asteroid Detector
A buddy module to Photometry Pipeline for asteroid detection and reporting


<img src="http://www.rankinstudio.com/dnloads/AsteroidDetector.JPG" width="600px" />

<center><b>INTRODUCTION:</b></center>

This is a guide on how you can get Asteroid Detector up and running. Just a fair warning, this will be rather long and complicated. If you are not familiar with Linux or Python, it may prove to be difficult to get working. It is a personal project I built for my own needs and am offering free without support. If you find any massive bugs or errors in the guide please let me know. You need an MPC observatory code to submit observations of asteroids.

Asteroid detector works by using Photometerypipeline (PP) to extract sources from your images, and calibrate them. It works by stacking groups of images to get better S/N ratios then looking in the stacked images for asteroids, it also works with single images. These sources are stored in .db files. Asteroid Detector scans these .db files for moving objects based on some search parameters you provide. If found, it uses DS9 to plot these asteroids for you to confirm if they are real or not with a simple y or n in the command line. It will completely build your MPC report files. It can also parse the MPC to identify asteroids, and has the MPC's object rating tool (Digest2) built in so you can see the orbit probabilities as you scan through the detections. Asteroid Detector and PP create a lot of files during calibration and object searching. You can clean most of the unused files up from the GUI. The MPC report file (Report.txt) is saved in whatever directory you are working in. Asteroid Detector assumes you have somewhat accurate RA / DEC information in your fits headers for plate solving.

<center><b>SETUP:</b></center>
A Linux installation, I'm running Ubuntu 16.04 in VirtualBox
I setup a shared folder between Windows and VirtualBox to move files into Linux

Install Source Extractor, SWarp, and Scamp

sudo apt-get update
sudo apt-get install sextractor
sudo apt-get install scamp
sudo apt-get install swarp

I highly recommend reading over the documentation for Source Extractor, SWarp, and Scamp to get a basic idea of how they work together.

The next step is to get Photometrypipeline (PP) running: <a href="http://photometrypipeline.readthedocs.io/en/latest/install.html" target="_blank">PhotometeryPipeline</a>
There are detailed instructions at that site for getting PP up and going.
Install PP in /home/YourUsername/photometerypipeline/

To get SWarp running right from PP, I had to modify the pp_combine file like so:

In pp_combine line 234:
    commandline = (('swarp -combine Y -combine_type %s -delete_tmpfiles '+
change to:
    commandline = (('SWarp -combine Y -combine_type %s -delete_tmpfiles '+


From there, you'll have to make some changes to your bashrc file
Edit the bashrc from terminal with nano

nano ~/.bashrc


append these lines to the end, replace username with your username in Linux

# photometry pipeline setup
export PHOTPIPEDIR=/home/username/photometrypipeline/
export PATH=$PATH:/home/username/photometrypipeline/
export PATH=$PATH:/home/username/AsteroidDetector/digest2/
alias python=python3

This makes files in these folders available to run anywhere from the terminal window

To test this, you should be able to open a terminal and just run a command like pp_run or pp_combine. It should prompt with options you are missing.

david@david-VirtualBox:~$ pp_run
usage: pp_run [-h] [-prefix PREFIX] [-target TARGET] [-filter FILTER]
              [-fixed_aprad FIXED_APRAD]
              [-source_tolerance {none,low,medium,high}] [-solar]
              images [images ...]
pp_run: error: too few arguments

If you do not get this result, PP is not installed correctly, or not referenced correctly in the bashrc file.

PP requires specific configuration for your telescope equipment. The documentation for this is on the page above for PP. If you need help, here is an example of my "mytelescope.py" file:
This is made by copying the provided "telescopes.py" file and making modifications to match the information in your image fits headers. If you compare the differences between the file below and the provided "telescopes.py" file you wills see the changes I had to make to get the software working.

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
[/text]

You'll also notice that the YOURCCD.swarp, YOURCCD.scamp, and YOURCCD.sex files are referenced as well. Copy them from another scope in the PP setup directory and rename them to match the paths you provide in the "mytelescope.py" file above. There is a lot of other information in here you have to fill out from your CCD header. Just read it over and you'll figure out what goes where.

I changed the photometery catalogs to only " 'photometry_catalogs': ['APASS9']" as I was having issues getting the others to work well. APASS9 seems to work good for calibrating magnitudes in your images. You can commend that line and uncomment the one above to use the other two catalogs. 

You'll now need to install DS9. <a href="http://ds9.si.edu/site/Download.html" target="_blank">DS9 Download Page</a>
This is another program you'll want to get familiar with. I extracted DS9 to /usr/local/bin/ so it could be ran from the command line anywhere. You can put it anywhere if you modify your bashrc file with the PATH like we did above. Make sure you can launch DS9 from a terminal window by simply typing DS9. 

There are a lot of python modules that Asteroid Detector relies on. Most of them come with Ubuntu 16.04 but some you'll have to install with apt-get or pip
Make sure all of these Python3 modules are installed: pyds9, pickle, glob, datetime, astropy, bs4, subprocess, shlex, tkinter, requests, random, string, sqlite3, numpy

From here, you should do some extractions using pp_extract and change your .sex file to output a "check.fit" image of APERTURES so you can see how your extraction settings are working.

<center><b>Asteroid Detector</b></center>
Asteroid Detector expects data in a certain directory structure:
[text]
Folder(Sequencename)
   (Folder1)
      -Image1
      -Image2
      ect..
   (Folder2)
      -Image1
      -Image2
      ect...
   (Folder3)
   (Folder4)
[/text]
It also assumes these stacks are evenly spaced in time, so that a moving object will have somewhat even gaps between positions. This should be a standard searching method. As mentioned above, it stacks 4 groups of images to achieve better S/N ratio for searching. With my setup, I typically shoot 35 X 40 sec subs, then group them in stacks of 5. So images 1-5, 11-15, 21-25, 31-35. 

After you verify that all the above is working, getting Asteroid Detector installed is easy. Just download this archive and extract to /home/YourUsername/AsteroidDetector
Download Asteroid Detector: <a href="/dnloads/AsteroidDetector.tar.gz" target="_blank"> Asteroid Detector </a> 

In the Asteroid Detector directory you'll see a desigused.txt file. This keeps track of the designations you send to the MPC so it doesn't create duplicates. 

There is also an obsconfig.txt file. This contains the settings for Asteroid Detector:
[text]
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

### Defaults ArcSec ###
Default FWHM Minimum: 1.5
Default Asteroid Search Radius: 17
Default Star Search Radius: 1
Default Limiting Mag: 21
Max Resid: 0.35
Spacing Res: 14
Default File Open Dir: /home/david/Desktop

### DS9 Settings ###
DS9 Text Color: green
Invert images?: no
Image Keyword: Light
Blink Inverval: 0.2
DS9 Font Size: 15
[/text]
The settings are pretty self explanatory. Reference DS9 manual for options with color and blinking interval ect.
Two important settings are "Max Resid" and "Spacing Res". Resid will toss out objects over the given threshold, and spacing res has to do with how unevenly spaced the points can be along the line. If you increase this, it is more strict. So a lower value will allow more unevenly spaced points. This helps weed out false detections. 

To make a quicklaunch for it, create a new file on your Desktop called "AsteroidDetector.sh" or whatever you'd like. In this file put:
[text]
gnome-terminal -e "python3 /home/YourUsername/AsteroidDetector/AsteroidGUI.py"
[/text]
Right click on the file, go to properties, and make sure it is set to executable under permissions. 

<center><b>Using Asteroid Detector</b></center>

From here, you just launch it and select your working directory with the structure specified above. Click on "PROCESS IMGS". Be patient, this can take a while

Once this completes and you have good extractions in the .db files, you don't need to run it again. You can change your asteroid searching parameters and run any number of searches on the .db files you'd like without having to re-register the images.

If this completes successfully you'll get a message to search for asteroids, you can then proceed with that.  Output should show the number of possible asteroids found along with a prompt to search the MPC for matches as you go: 
[text]
0.00417 Asteroid search radius in degrees

Searching for asteroids images 1-3, and 1-4

Searching for asteroids images 2-4

#########################################
#####  4  Possible asteroids found  #####
#########################################

Search for designations?
[/text]
After this, it should launch DS9 and step you through the possible asteroids 1 by 1. If you answer yes to any of them and you chose to search MPC you should get output like this:
[text]
Searching MPC, please wait..

Information for TMP001
##########################################
#####    MPC SEARCH RESULTS BELOW    #####
##########################################

              Object designation         R.A.      Decl.     V       Offsets     Motion/hr   Orbit  Further observations?
                                        h  m  s     °  '  "        R.A.   Decl.  R.A.  Decl.        Comment (Elong/Decl/V at date 1)
     TMP001  P1972                    06 10 00.7 +27 35 11  18.9   0.0E   0.0N    28-    10-   10o  None needed at this time.
 
 Number of objects checked =  792526
 
############################################
#####    OBJECT PROBABILITIES BELOW    #####
############################################
Desig.    RMS Int NEO N22 N18 Other Possibilities
TMP001   0.24   2   2   1   0 (MC 1) (MB1 39) (Pal <1) (Han <1) (MB2 35) (MB3 21) (Hil <1) (JFC <1)
[/text]

Asteroid Detector parsed the MPC and found a match for the asteroid, and it also displayed the orbit probabilities for the asteroid. If you chose not to overwrite an existing report, it doesn't write any information out and assigns a temp designation of TMP001 for all rocks.

<center><b>Advanced Configuration</b></center>

Once this is all working, you need to verify you are getting good extraction from source extractor, and that your residuals are good across the frame. My images are warped pretty bad by the optics because my FOV is pretty large. I had to change the setting in my PP/setup/MYSCOPE.scamp file: DISTORT_DEGREES to 3 in order to account for this. Getting somewhat familiar with Source Extractor, Scamp, and SWarp will help you a lot.

You can access the two most common Source Extractor settings in the "mytelescope.py" config file above. You can see what I chose for mine.

This has been a lot of fun working on and in the long run will save me A LOT of time loading images, solving them, stacking them, and then searching for asteroids. If you can get it running I hope you also find it useful.

I'll be making changes to it a lot. Check back often.

Upcoming changes: PA and VEL astrometry for faster objects.
