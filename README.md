dejavu
==========

Audio fingerprinting and recognition algorithm implemented in Python, see the explanation here:  
[How it works](http://willdrevo.com/fingerprinting-and-audio-recognition-with-python.html)

Dejavu can memorize audio by listening to it once and fingerprinting it. Then by playing a song and recording microphone input, Dejavu attempts to match the audio against the fingerprints held in the database, returning the song being played. 

Note that for voice recognition, Dejavu is not the right tool! Dejavu excels at recognition of exact signals with reasonable amounts of noise.

## Installation and Dependencies:

# Installation of Dejavu

So far Dejavu has only been tested on Unix systems.

* [`pyaudio`](http://people.csail.mit.edu/hubert/pyaudio/) for grabbing audio from microphone
* [`ffmpeg`](https://github.com/FFmpeg/FFmpeg) for converting audio files to .wav format
* [`pydub`](http://pydub.com/), a Python `ffmpeg` wrapper
* [`numpy`](http://www.numpy.org/) for taking the FFT of audio signals
* [`scipy`](http://www.scipy.org/), used in peak finding algorithms
* [`matplotlib`](http://matplotlib.org/), used for spectrograms and plotting
* [`MySQLdb`](http://mysql-python.sourceforge.net/MySQLdb.html) for interfacing with MySQL databases

For installing `ffmpeg` on Mac OS X, I highly recommend [this post](http://jungels.net/articles/ffmpeg-howto.html).

## Fedora 20+

### Dependency installation on Fedora 20+

Install the dependencies:

    sudo yum install numpy scipy python-matplotlib ffmpeg portaudio-devel MySQL-python
    pip install PyAudio
    pip install pydub
    
Now setup virtualenv ([howto?](http://www.pythoncentral.io/how-to-install-virtualenv-python/)):

    pip install virtualenv
    virtualenv --system-site-packages env_with_system

Install from PyPI:

    source env_with_system/bin/activate
    pip install PyDejavu


You can also install the latest code from GitHub:

    source env_with_system/bin/activate
    pip install https://github.com/worldveil/dejavu/zipball/master

## Max OS X

### Dependency installation for Mac OS X

Tested on OS X Mavericks. An option is to install [Homebrew](http://brew.sh) and do the following:

```
brew install portaudio
brew install ffmpeg

sudo easy_install pyaudio
sudo easy_install pydub
sudo easy_install numpy
sudo easy_install scipy
sudo easy_install matplotlib
sudo easy_install pip

sudo pip install MySQL-python

sudo ln -s /usr/local/mysql/lib/libmysqlclient.18.dylib /usr/lib/libmysqlclient.18.dylib
```

However installing `portaudio` and/or `ffmpeg` from source is also doable. 

## Setup

First, install the above dependencies. 

Second, you'll need to create a MySQL database where Dejavu can store fingerprints. For example, on your local setup:
	
	$ mysql -u root -p
	Enter password: **********
	mysql> CREATE DATABASE dejavu DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_unicode_ci;

Now you're ready to start fingerprinting your audio collection! 

## Quickstart

```bash
$ git clone https://github.com/worldveil/dejavu.git ./dejavu
$ cd dejavu
$ python example.py
```

## Fingerprinting

Let's say we want to fingerprint all of July 2013's VA US Top 40 hits. 

Start by creating a Dejavu object with your configurations settings (Dejavu takes an ordinary Python dictionary for the settings).

```python
>>> from dejavu import Dejavu
>>> config = {
...     "database": {
...         "host": "127.0.0.1",
...         "user": "root",
...         "passwd": <password above>, 
...         "db": <name of the database you created above>,
...     }
... }
>>> djv = Dejavu(config)
```

Next, give the `fingerprint_directory` method three arguments:
* input directory to look for audio files
* audio extensions to look for in the input directory
* number of processes (optional)

```python
>>> djv.fingerprint_directory("va_us_top_40/mp3", [".mp3"], 3)
```

For a large amount of files, this will take a while. However, Dejavu is robust enough you can kill and restart without affecting progress: Dejavu remembers which songs it fingerprinted and converted and which it didn't, and so won't repeat itself. 

You'll have a lot of fingerprints once it completes a large folder of mp3s:
```python
>>> print djv.db.get_num_fingerprints()
5442376
```

Also, any subsequent calls to `fingerprint_file` or `fingerprint_directory` will fingerprint and add those songs to the database as well. It's meant to simulate a system where as new songs are released, they are fingerprinted and added to the database seemlessly without stopping the system. 

## Configuration options

The configuration object to the Dejavu constructor must be a dictionary. 

The following keys are mandatory:

* `database`, with a value as a dictionary with keys that the database you are using will accept. For example with MySQL, the keys must can be anything that the [`MySQLdb.connect()`](http://mysql-python.sourceforge.net/MySQLdb.html) function will accept. 

The following keys are optional:

* `fingerprint_limit`: allows you to control how many seconds of each audio file to fingerprint. Leaving out this key, or alternatively using `-1` and `None` will cause Dejavu to fingerprint the entire audio file. Default value is `None`.
* `database_type`: as of now, only `mysql` (the default value) is supported. If you'd like to subclass `Database` and add another, please fork and send a pull request!

An example configuration is as follows:

```python
>>> from dejavu import Dejavu
>>> config = {
...     "database": {
...         "host": "127.0.0.1",
...         "user": "root",
...         "passwd": "Password123", 
...         "db": "dejavu_db",
...     },
...     "database_type" : "mysql",
...     "fingerprint_limit" : 10
... }
>>> djv = Dejavu(config)
```

## Tuning

Inside `fingerprint.py`, you may want to adjust following parameters (some values are given below).

    FINGERPRINT_REDUCTION = 30
    PEAK_SORT = False
    DEFAULT_OVERLAP_RATIO = 0.4
    DEFAULT_FAN_VALUE = 10
    DEFAULT_AMP_MIN = 15
    PEAK_NEIGHBORHOOD_SIZE = 30
    
These parameters are described in the `fingerprint.py` in detail. Read that in-order to understand the impact of changing these values.

## Recognizing

There are two ways to recognize audio using Dejavu. You can recognize by reading and processing files on disk, or through your computer's microphone.

### Recognizing: On Disk

Through the terminal:

```bash
$ python dejavu.py --recognize file sometrack.wav 
{'song_id': 1, 'song_name': 'Taylor Swift - Shake It Off', 'confidence': 3948, 'offset_seconds': 30.00018, 'match_time': 0.7159781455993652, 'offset': 646L}
```

or in scripting, assuming you've already instantiated a Dejavu object: 

```python
>>> from dejavu.recognize import FileRecognizer
>>> song = djv.recognize(FileRecognizer, "va_us_top_40/wav/Mirrors - Justin Timberlake.wav")
```

### Recognizing: Through a Microphone

With scripting:

```python
>>> from dejavu.recognize import MicrophoneRecognizer
>>> song = djv.recognize(MicrophoneRecognizer, seconds=10) # Defaults to 10 seconds.
```

and with the command line script, you specify the number of seconds to listen:

```bash
$ python dejavu.py --recognize mic 10
```

## Testing

Testing out different parameterizations of the fingerprinting algorithm is often useful as the corpus becomes larger and larger, and inevitable tradeoffs between speed and accuracy come into play. 

Test your Dejavu settings on a corpus of audio files on a number of different metrics:

* Confidence of match (number fingerprints aligned)
* Offset matching accuracy
* Song matching accuracy
* Time to match

An example script is given in `test_dejavu.sh`, shown below:

```bash
#####################################
### Dejavu example testing script ###
#####################################

###########
# Clear out previous results
rm -rf ./results ./temp_audio

###########
# Fingerprint files of extension mp3 in the ./mp3 folder
python dejavu.py --fingerprint ./mp3/ mp3

##########
# Run a test suite on the ./mp3 folder by extracting 1, 2, 3, 4, and 5 
# second clips sampled randomly from within each song 8 seconds 
# away from start or end, sampling offset with random seed = 42, and finally, 
# store results in ./results and log to ./results/dejavu-test.log
python run_tests.py \
    --secs 5 \
    --temp ./temp_audio \
    --log-file ./results/dejavu-test.log \
    --padding 8 \
    --seed 42 \
    --results ./results \
    ./mp3
```
