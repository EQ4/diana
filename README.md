#DIANA#
* Dynamic Interactive Audio and Noise Analyzer
* Uri Nieto
* 2009-14
* New York University
* Stanford University

#REQUIREMENTS#

* Mac OS X 10.4 or higher
* PortAudio (http://www.portaudio.com/)

#BUILDING THE PROJECT#

From the directory where the files of the tgz are, type the following:

    make

This should compile and generate the Diana binary.

#USAGE#

To run `Diana' type (from the `Diana' directory):

    ./Diana

These are the different possible options while running Diana:

    'h' - print help message
    'f' - toggle fullscreen
    'k' - toggle signal / keyboard mode
    'CURSOR ARROWS' - rotate signal view
    'q' - quit

#DESCRIPTION#

Diana is a small piece of software that analyzes noise and audio
and displays it in a nice 3d window. It also estimates the pitch
from a relatively stable sinusoidal signal and maps it to
a midi keyboard key in the screen.

Diana stands for Dynamic Interactive Audio and Noise Analyzer.
It uses PortAudio for the input signal, chuck libraries for
the FFT and OpenGL for the graphics. The code is based on 
Sound Peek by Ge Wang, Perry R. Cook, Ananya Misra
http://soundlab.cs.princeton.edu/

At the moment, Diana only runs on Mac OS X 10.4 or higher.

Diana has 2 different running modes: Signal and Keyboard

##Signal Mode##

This is the default mode, and it prints 4 different signals on
the screen, all of them being read from your default input.

Top Green signal: Windowed Time-Domain Signal. The window used
is a Hanning window.

Middle Blue signal: Scrolling Time-Domain Signal.

Bottom Orange signal: Waterfall Spectrum of the Windowed Time-
Domain Signal.

Background Blue signal: Time-Domain signal being rotated, and no
window applied. Also called Speed of Light signal.

You can rotate this view by pressing any cursor arrows. All
signals will rotate except the one in the back, which will stay
still creating this "speed of light" effect.

##Keyboard Mode##

This mode shows a midi keyboard in the screen and it will 
estimate the pitch from the signal being read and map it to the
correct keyboard key. There is a deviation error of +1/-1 semitone.

To toggle between Signal and Keyboard mode, press 'k'.

To toggle between Fullscreen mode press 'f'.

By default, the Signal Mode is on and the fullscreen is off.

#IMPLEMENTATION#

The tough part of the implementation is based on the graphics to
display the signal read by the default input. However, it is
important to remind how we read from the default input. We use
PortAudio in order to do that, and in this case, the audio callback
is a simple function that stores the input buffer into our buffer
that will read our display function.

The only "tricky" part is that we use the mutex options to lock the 
thread, so that it will be safe to read from that buffer in the 
display function.

Once we have our buffer ready to be read by the display function,
all the action is focused on this function. The display function is
the callback function of the OpenGL library, and it will be called
every time it needs to refresh the screen.

We can divide the implementation of the display function into
5 different parts: Windowed Time-Domain, Rotating Time-Domain,
Scrolling Time-Domain, Spectrum Waterfall, Pitch Detection.

##Windowed Time-Domain##

In order to display the windowed time domain (the green signal in
the top of the screen), we must apply the window first to our
buffer where the signal is stored. We must first define our window
type. This will only be done once, and that is why it is found in 
the main function. Here is the code to do that:

    // make the transform window
    hanning( g_window, g_buffer_size );

The hanning function is found in the chuck_fft code, extracted
from sndpeek. The window type then, is the hanning window.

Now that we have defined our window, we can apply it to our
signal, this will be done in the display function, so every time
the screen is refreshed:

    // apply the transform window
    apply_window( (float*)buffer, g_window, g_buffer_size );

This apply_window is also found in the chuck_fft code.

Now we can display this buffer in the top of the screen using basic
OpenGL syntax, applying the desired color. We will push the matrix
so that we can safe the state and don't mess up with the rest
of the signals.

The function to draw the windowed time-domain signal is:
void drawWindowedTimeDomain(SAMPLE *buffer);

##Rotating Time-Domain##

This signal is the one in the background, the one that creates
this sensation of "Speed of Light". In this case, we don't want 
to apply the window, so we will draw this signal before applying
the window to our buffer.

Moreover, since we want to keep this Rotating Time-Domain signal
in the background without rotations, we will do it before
saving the push matrix for rotations, so that it won't move when
rotating the rest of the signals.

In order to create this effect of "Speed of Light", we will rotate
the z axis at a fast and slightly random speed. The line width is
slightly higher and the signal is multiplied by a factor of 2 so that
the effect is stronger.

The function to draw the rotating time-domain signal is:
void drawRotatingTimeDomain(SAMPLE *buffer);

##Scrolling Time-Domain##

We will need a new buffer in order to store the information to be
displayed in the screen. This buffer will be much bigger, and the size
of it will determine how much of the previous samples we want to
display.

In our case, our buffer is:
SAMPLE g_scroll_buffer[DNA_SCROLL_BUFFER_SIZE];

where:
DNA_SCROLL_BUFFER_SIZE = DNA_BUFFER_SIZE*60

So it's 60 times bigger than the buffer size (which is 1024).

This buffer will be a "circular" buffer, and to do so we will use 
to different indices:

    int g_scroll_reader;
    int g_scroll_writer;

At every refresh of the screen, we will copy the input buffer into
our scrolling buffer in the position where the g_scroll_writer says.

This way we won't have to go through all the buffer in order to 
update the information displayed. Thus, is much more efficient.

We will draw the scrolling time-domain using standard OpenGL. The
code is found in this function:
void drawScrollingTimeDomain(SAMPLE *buffer);

##Spectrum Waterfall##

We will use a similar system to the one used in sound peek in order
to implement the spectrum waterfall. We will first of all apply the
FFT to our buffer. We will use the libraries from chuck_fft to do
so:

    // take forward FFT; result in buffer as FFT_SIZE/2 complex values
    rfft( (float *)buffer, g_fft_size/2, FFT_FORWARD );
        
    // cast to complex
    complex * cbuf = (complex *)buffer;

We cast to complex to compute the spectrum in an easy way.

We will make use of a global variable called g_spectrums, which
is an array of spectrums. We will make use of Pt2D, which is a 
point in 2D with the following structure:

    struct Pt2D { float x; float y; };

So our g_spectrums is:

    Pt2D ** g_spectrums;

We will first set our g_spectrums to zero, with a depth of 
48. Then, in every refresh, we will read from the complex buffer
and store the result into the correct position of our g_spectrums.

We will gradually change the color in order to produce a nice
visual effect of the waterfall.

The code in order to draw the spectrum waterfall is found in:
void drawSpectrumWt(complex *cbuf);

##Pitch Detection##

The pitch detection will be done before any rotation, like the
"Speed of Light", so that it won't rotate even though the rest
of the signals are rotated.

I used autocorrelation in order to obtain a better pitch estimation.
Then, I read the autocorrelated buffer and interpolate the max peak
and assign it to a real note. The real note has an error of +1/-1
semitone. This is done in the function:

    void getNote(double pitch);

In order to show the Midi Keyboard, I used the Textures options
from OpenGL. To load the textures, I implemented the following
function:

    GLuint LoadTextureRAW( const char * filename, int wrap );

The whole implementation in order to draw the midi keyboard
and get the pitch is found in:

    // Draw the Keyboard
    drawKeyboard();
    
    // Get the Pitch and assign to keyboard key
    getPitchAndAssignToKeyboard(buffer);

There is a small pitch stability algorithm that will only take
in consideration the pitches that are constant in 4 continuous
time-windowed buffers. This makes the pitch detection
much more stable.


The hardest part of this assignment was the OpenGL part, since
I hadn't played with it for a long time. Also took me some time
to deal with the scrolling buffer, and the pitch detection.

The other difficult part was to stay away from it, since in the
end I considered this project my "little son" and I couldn't stop
adding new features.


Have fun!
uri

oriol@nyu.edu

MARL

New York University

2014
