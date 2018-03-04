---
title: Data Compression Note
date: 2018-02-01 01:29:15
tags: note
---
# Data Compression

## Huffman Table
### Concept
1. sort by frequency
2. merge two smallest node until root node left
### Example
![](https://i.imgur.com/5fTk7MR.png)

![](https://i.imgur.com/Wpzqs8T.png)

![](https://i.imgur.com/SCSUi1C.png)


## Adaptive Huffman coding
### Concept
- Each time the tree is updated either by adding a new character or increment the frequency of occurrence of an existing character
- If there is a node out of order, the structure of the tree is modified by exchanging the position of this node with the other node in the tree
### Example
the following example is transmitting a stream `This-is`
*[] is empty*


init
![](http://i.imgur.com/5r3z4YA.png)
| index | character | output | list | in order? |
|-------|-----------|--------|------|-----------|
| 1     | T         | 0"T"   | []T1 | Y         |
![](http://i.imgur.com/tn5M6N6.png)
| index | character | output | list       | in order? |
|-------|-----------|--------|------------|-----------|
| 2     | h         | 0"h"   | [] h1 1 T1 | Y         |
![](http://i.imgur.com/DViIVg0.png)
| index | character | output | list                | in order? |
|-------|-----------|--------|---------------------|-----------|
| 3     | i         | 00"i"  | [] i1 1 h1 **2 T1** | N         |
![](http://i.imgur.com/ksInmzS.png)
| index | character | output | list            | in order? |
|-------|-----------|--------|-----------------|-----------|
| 3’    |           |        | [] i1 1 h1 T1 2 | Y         |
![](http://i.imgur.com/WLOrsQg.png)
| index | character | output | list                         | in order? |
|-------|-----------|--------|------------------------------|-----------|
| 4     | s         | 100"s" | [] s1 1 i1 **2** h1 **T1** 3 | N         |
*find the rightmost node which smaller than 2 to swap*
![](http://i.imgur.com/3kbfTNO.png)
| index | character | output | list                 | in order? |
|-------|-----------|--------|----------------------|-----------|
| 4’    |           |        | [] s1 1 i1 T1 h1 2 2 | Y         |
![](http://i.imgur.com/a73icG4.png)
| index | character | output | list                              | in order? |
|-------|-----------|--------|-----------------------------------|-----------|
| 5     | -         | 000"-" | [] -1 1 s1 **2** i1 T1 **h1** 3 2 | N         |
![](http://i.imgur.com/5iEpDF8.png)
| index | character | output | list                      | in order? |
|-------|-----------|--------|---------------------------|-----------|
| 5’    |           |        | [] -1 1 s1 h1 i1 T1 2 2 3 | Y         |
![](http://i.imgur.com/aTduejD.png)
| index | character | output | list                          | in order? |
|-------|-----------|--------|-------------------------------|-----------|
| 6     | i         | 01     | [] -1 1 s1 h1 **i2 T1** 2 3 3 | N         |
![](http://i.imgur.com/FIqjjm3.png)
| index | character | output | list                      | in order? |
|-------|-----------|--------|---------------------------|-----------|
| 6’    |           |        | [] -1 1 s1 h1 T1 i2 2 2 4 | Y         |
![](http://i.imgur.com/Fn26bYo.png)
| index | character | output | list                              | in order? |
|-------|-----------|--------|-----------------------------------|-----------|
| 7     | s         | 111    | [] -1 1 **s2** h1 **T1** i2 3 2 5 | N         |
![](http://i.imgur.com/pMK7h6f.png)
| index | character | output | list                      | in order? |
|-------|-----------|--------|---------------------------|-----------|
| 7’    |           |        | [] -1 1 T1 h1 s2 i2 2 3 4 | Y         |
![](http://i.imgur.com/77IFLXD.png)

## Arithmetic coding
### Concept
- Represent a sequence of symbols by an interval with length equal to its probability
- The interval is specified by its lower boundary (l), upper boundary (u) and length d (=probability)
- The codeword for the sequence is the common bits in binary representations of l and u
- A more likely sequence=a longer interval=fewer bits
- each symbol (n with probability p~n~) is located in a interval among (0,1)
- q~n-1~ is the sum of probability of previous symbols

### Encoding Process
- initial values l~0~=0 u~0~=1 d~0~=1

- For n^th^ symbol
    with q~n-1~ and p~n~ at l~n-1~,u~n-1~,d~n-1~=u~n-1~-l~n-1~

    - d~n~ = d~n-1~ * p~n~
    - l~n~ = l~n-1~ + d~n-1~ * q~n-1~
    - u~n~ = l~n~ + d~n~

#### Example
p~a~=1/2
p~b~=1/4
p~c~=1/4

q~a~=0
q~b~=p~a~=1/2
q~c~=p~a~+p~b~=3/4
For symbol stream `abaca` :
| symbol | q   | p   | l                  | u                  | d              |
|--------|-----|-----|--------------------|--------------------|----------------|
|        |     |     | 0                  | 1                  | 1              |
| a      | 0   | 1/2 | 0=0+1x0            | 1/2=0+1/2          | 1/2=1x1/2      |
| b      | 1/2 | 1/4 | 1/4=0+1/2x1/2      | 3/8=1/4+1/8        | 1/8=1/2x1/4    |
| a      | 0   | 1/2 | 1/4=1/4+1/8x0      | 5/16=1/4+1/16      | 1/16=1/8x1/2   |
| c      | 3/4 | 1/4 | 19/64=1/4+1/16x3/4 | 20/64=19/64+1/64   | 1/64=1/16x1/4  |
| a      | 0   | 1/2 | 19/64=19/64+1/64x0 | 39/128=19/64+1/128 | 1/128=1/64x1/2 |

Output = value of **u** =39/128=0.0100111 (binary)

![](https://i.imgur.com/kp9Owq4.png)

### Decoding Process
- with input u and symbol interval in (0,1)
- 1.Push u into (0,1), and check which interval it is located, output the symbol own this interval.
  2.u = u – low boundary of this interval
  3.u = u / the range of this interval, ignore the carry
  4.repeat step 1~3 till u is 0

#### Example
u is 0.0100111, with interval a(0,1/2], b(1/2,3/4], c(3/4,1]
| u      | interval  | 1/range | symbol | new u |
|--------|-----------|---------|--------|-------|
| 39/128 | (0,1/2]   | 2       | a      | 39/64 |
| 39/64  | (1/2,3/4] | 4       | b      | 7/16  |
| 7/16   | (0,1/2]   | 2       | a      | 7/8   |
| 7/8    | (3/4,1]   | 4       | c      | 1/2   |
| 1/2    | (0,1/2]   | 2       | a      | 0     |

Output:abaca

## LZW
### Property
- input:variable length output:fixed length
- dictionary based compression
    - winzip
    - winarj
- Using dictionary table that map character ( word, or character stream) to index (fixed bits)
    | index | New stream       |
    |-------|------------------|
    | 0     | prefix+character |
    | 1     | prefix+character |
    | .     | .                |
    | .     | .                |
    | n     | prefix+character |
- length of index, which dependent the size of dictionary, is the output of encoding
### Encoding Process
1. Define the size of DIC (dictionary) with 12 or 16 bits. Here is 12bits example
2. Initialize the dictionary with the first 128 ASCII code, 000~07f
3.
    (1)Set NS (new stream) as empty
    (2)Input NC (new character) check if NS+NC is in dictionary
    - yes, NS=NS+NC, repeat (2)
    - No, output the index of NS in dictionary append NS+NC into dictionary as new word NS=NC, repeat (2) till end of file (input)

#### Example
A dictionary with root characters `ABCD`, encode `ABCABCDABCABCAD` Initial dictionary with 12 bits Dictionary
<000> = “?”
...
<041> = “A”
<042> = “B”
<043> = “C”
<044> = “D”
...
<07f>  = “?”


|    Input   (NC)     |    dictionary      |    new   stream    |    output         |
|---------------------|--------------------|--------------------|-------------------|
|                     |                    |    [ ]             |                   |
|    A                |    ASCII   code    |    [A]             |                   |
|    B                |    AB>>080         |    [B]             |    041,   A       |
|    C                |    BC>>081         |    [C]             |    042,   B       |
|    A                |    CA>>082         |    [A]             |    043,   C       |
|    B                |                    |    [AB]            |                   |
|    C                |    ABC>>083        |    [C]             |    080,   AB      |
|    D                |    CD>>084         |    [D]             |    043,   C       |
|    A                |    DA>>085         |    [A]             |    044,   D       |
|    B                |                    |    [AB]            |                   |
|    C                |                    |    [ABC]           |                   |
|    A                |    ABCA>>086       |    [A]             |    083,   ABC     |
|    B                |                    |    [AB]            |                   |
|    C                |                    |    [ABC]           |                   |
|    A                |                    |    [ABCA]          |                   |
|    D                |    ABCAD>>087      |    [D]             |    086,   ABCA    |
|    End   of file    |                    |                    |    044,   D       |

### Decoding Process
*NC: new code, NW: new word in dictionary*
1. Input first NC, NW=NC, output character of NC
2. Input NC Check NC, if NC is in dictionary?
    - Yes, set NW=NW+ first character in NC
    - No. set NW=NW+ first character in NW
3. Put NW into dictionary as new word, NW=NC
4. Output the character stream of NC Is not EOF?, yes, repeat 1;otherwise, end decoding
#### Example
|    Input   code     |    New   word    |    dictionary      |    Output   stream    |
|---------------------|------------------|--------------------|-----------------------|
|                     |                  |    ASCII   code    |                       |
|    041              |                  |                    |    A                  |
|    042              |    041+042       |    AB>>080         |    B                  |
|    043              |    042+043       |    BC>>081         |    C                  |
|    080              |    043+080       |    CA>>082         |    AB                 |
|    043              |    080+043       |    ABC>>083        |    C                  |
|    044              |    043+044       |    CD>>084         |    D                  |
|    083              |    044+083       |    DA>>085         |    ABC                |
|    086              |    083+086       |    ABCA>>086       |    ABCA               |
|    044              |    086+044       |    ABCAD>>087      |    D                  |
|    End   of file    |                  |                    |                       |



## Sound
### Waveform coding
- Sample and code
- High-quality and not complex
- Large amount of bandwidth
#### PCM(Pulse-code modulation)
- A method used to digitally represent sampled analog signals. It is the standard form of digital audio in computers, compact discs, digital telephony and other digital audio applications
- Not really a compression technique

![](https://i.imgur.com/UoMWlKU.png)


#### DPCM(Differential PCM)
- Only transmit the difference between the predicated value and the actual value
- It is possible to predict the value of a sample base on the values of previous samples
- The receiver perform the same prediction
- No algorithmic delay
- EX: Voice changes relatively slowly
- Take PCM sequence `250 268 269 241` for example, by using DPCM to improve, we can store the sequence into `250 +18 +19 -9` to save more space but doesn't change the data, so DPCM is **Lossless Compression**.
![](https://i.imgur.com/BVxrFkH.png)

#### ADPCM(Adaptive Differential PCM)
- Quantization level varies with local signal level
- locally estimated standard deviation
- From the example `250 +18 +19 -9` above,by using ADPCM, the sequence can be simplified as `250 (+9,+9,-4)x2` to save more space.But when decoding:`(+9,+9,-4)x2=250, +18 +18 -8`,the data is different from original source, so DPCM is **Lossy Compression**.
### Source coding
- Based on a model of how the human voice is produced
- The waveform is discarded and parameters describing the characteristics of the sounds produced are transmitted instead. (Linear Predictive Coding)
- The synthesised voice tends to be rather artificial
- Generic models are used to reproduce vowels (vibrations of the vocal chords) and consonants (produced by shaping the mouth).
- Vocal Tract Model(不用記)
$$G(z)={1 \over 1-\sum_{k=1}^Pa_kZ^{-k}}$$
*a~k~ : LPC coefficients*
#### LPC(Linear Prediction)
Linear prediction is at the base of many speech coding techniques, including CELP. The idea behind it is to predict the signal using a linear combination of its past samples
#### CELP(Code-excited linear prediction)[^1][^2][^3]
:::info
**Vocal cords** are the source of spectrally flat sound (the excitation signal)
:::
:::info
**Vocal tract** acts as a filter to spectrally shape the excitation signal into various sounds.
:::
- Use LPC to model the vocal tract
- Use Pitch Prediction: during voiced segments, the speech signal is periodic, so it is possible to take advantage of that property by approximating the excitation signal by a gain times the past of the excitation
- Use adaptive and fixed codebook entries as input (excitation) of the LP model
- The search performed in closed-loop in a "perceptually weighted domain"
- Analysis-by-Synthesis
    The encoding (analysis) is performed by perceptually optimising the decoded (synthesis) signal in a closed loop. In theory, the best CELP stream would be produced by trying all possible bit combinations and selecting the one that produces the best-sounding decoded signal.

[^1]:[Introduction to CELP Coding](https://speex.org/docs/manual/speex-manual/node9.html)
[^2]:[Codetopic-excited linear predictive (CELP) coders (VoIP Protocols)](http://what-when-how.com/voip-protocols/codetopic-excited-linear-predictive-celp-coders-voip-protocols/)
[^3]:[CELP語音編碼器框架遺失訊號重整之研究](http://www.csie.ntu.edu.tw/~r95162/1.pdf)


![](https://i.imgur.com/rYG6JO2.png)

## MP3
### Absolute threshold of hearing
- is the minimum sound level of a pure tone that an average human ear with normal hearing can hear with no other sound present
![](https://i.imgur.com/PqCpDVU.png)

### Simultaneous masking
occurs when a sound is made inaudible by a noise or unwanted sound of the same duration as the original sound.For example, a powerful spike at 1 kHz will tend to mask out a lower-level tone at 1.1 kHz.
![](https://i.imgur.com/Pv39Qwa.png)

### Temporal Masking
occurs when a sudden stimulus sound makes inaudible other sounds which are present immediately preceding or following the stimulus
![](https://i.imgur.com/smMaiwy.png)

:::info
MP3 is based on the strength of the audio signal to achieve masking effect, thereby reducing the signal redundancy, to achieve the effect of compression
:::

## Run-length coding
- lossless data compression
- *runs* of data (that is, sequences in which the same data value occurs in many consecutive data elements) are stored as a single data value and count
- most useful on data that contains many such runs, and vice versa
### Example
`WWWWWWWWWWWWBWWWWWWWWWWWWBBBWWWWWWWWWWWWWWWWWWWWWWWWBWWWWWWWWWWWWWW`
encode to
`12W1B12W3B24W1B14W`

- When using for compressing binary image, it can be combined with huffman table

![](https://i.imgur.com/Jx7tBFC.png)
![](https://i.imgur.com/xgvPX1P.png)
![](https://i.imgur.com/4ZBeApt.png)


## READ coding
- Relative Element Address Designate coding
- Code the location of run boundary relative to the previous row.
![](https://i.imgur.com/3UobDi9.png)
### Pass mode
- Condition: (b 1 b 2 ) is to the left of (a 1 a 2 ) and b 2 is strictly to the left of a 1
- Run encoded: (b 1 b 2 )
- Codeword (P): 0001 + coded length of (b 1 b 2 )
- Update: a 0 = b 2 & the rest according to definitions
- ![](https://i.imgur.com/JwDuRl4.png)

### Horizontal mode
- Condition: (b 1 b 2 ) overlaps (a 1 a 2 ) by more than 3 pels as determined by |a 1 -b 1 |
- Runs encoded: (a 0 a 1 ), (a 1 a 2 )
- Codeword (H): 001 + coded lengths of (a 0 a 1 ), (a 1 a 2 )
- Update: a 0 = a 2 & the rest according to definitions
- ![](https://i.imgur.com/49NzOD4.png)

### Vertical mode
- Condition: (b 1 b 2 ) overlaps (a 1 a 2 ) by no more than 3 pels
- Expected as most common (assuming small differences)
- Run encoded: (a 1 b 1 )
- Codeword:
    - -3/-2/-1/0/1/2/3 → VR(3)/VR(2)/VR(1)/V(0)/VL(1)/VL(2)/VL(3)
    - 1/000001/00001/011/1/010/000010/0000010
- Update: a 0 = a 1 & the rest according to definitions
- ![](https://i.imgur.com/3cLc8V6.png)

![](https://i.imgur.com/pnO9AfO.png)

## JPEG

1. Transform RGB to YUV,  U and V down-sample 2x2.
![](https://i.imgur.com/lUUNnWT.png)
2. Blocking YUV into 8x8 blocks.Encoding blocks from right to left , top to down, Y, U and V in every block
![](https://i.imgur.com/zoOgY3T.png)
3. DCT transform into frequency domain.
![](https://i.imgur.com/55SpOkB.png)
4. Quantize the DCT coefficients with default quantization table.
- Different weighting matrices are standardized, adapted to human visual contrast sensitivity.
- DC coefficient is coded with DPCM using previous block’s DC as predictor,  then quantized.
![](https://i.imgur.com/td7EdS3.png)
5. Run length coding as run length symbol with zig-zag order from DC to maximal freqency. EOB is the last symbol. Run length symbol (R,A) = ( zero count, non-zero value ).
![](https://i.imgur.com/DS33mo1.png)
![](https://i.imgur.com/ssdWTJ7.png)
6. Entropy coding the run length symbols
![](https://i.imgur.com/PrzIzEe.png)
7. Use Huffman coding to compress data from entropy coding

## JPEG2000
### Why need JPEG2000
- For the users with different request in image quality or transmit bandwidth. In JPEG, many versions are coded and transmitted.
- For the users that are interested only one part in an image. (range of interest ROI)
- For the JPEG file with one part error.
- Scalable from lossless to high compressed in one compressed file.
- In order to address areas that the current standards fail to produce the quality or performance, as for example:
    - Low bit-rate compression: For example below 0.25bpp
    - Lossless and lossy compression: No current standard exists that can provide superiror lossy and lossless compression in a single codestream.
    - Computer generated imagery: JPEG was optimized for natural imagery and does not perform well on computer generated imagery.
    - Transmission in noisy environments: The current JPEG standard has provision for restart intervals, but image quality suffers dramatically when bit errors are encountered.
    - Compound documents: Currentyly, JPEG is seldom used in the compression of compound documents because of its poor perfornamce when applied to bi-level(text) imagery.
    - Open architecture
    - Progressive transmission by pixel accuracy and resolution
### Difference between JPEG
- New fuctionalities
    - Region Of Interest
    - Error resilience
    - Progression orders
- Lossy to lossless in one system
- Better compression at low bit-rates
- Better at compound images and graphics
### Codec
![](http://imgur.com/t1RNbDz.png)
### Image Coding System
![](http://i.imgur.com/BD3Oj5P.png)
![](http://i.imgur.com/g4nG391.png)

## MPEG
- Audio/video on CD-ROM
- Prompted explosion of digital video applications: MPEG1 video CD and downloadable video over Internet
- MPEG-1 Audio
    - Offers 3 coding options (3 layers), higher layer have higher coding efficiency with more computations
    - MP3 = MPEG1 layer 3 audio

### Frame
![](https://i.imgur.com/6SOkKvM.png)
- I-frames
    - No temporal redundacy reduction
    - Has the highest bit count
    - For random access, FF, REW features
- P-frames
    - Forward motion-compensated prediction
- B-frames
    - Both forward and backward motion-compensated prediction
    - Usually results in the lowest bit count
    - Increase delay
### Prediction Mode
In P frame, the macro-block (2x2 blocks) is estimated that may move from M-block in previous frame (I or Previous P), the displacement of M-blocks between previous one and current one is the Motion Vector (MVx, MVy), the current M-B will be replaced (predicted) by the previous one, and the difference will be coded with JPEG ( Motion Compensation).
![](https://i.imgur.com/IkOrRNG.png)
### Bi-direction Mode
In B frame, the M-B will be forward estimated from previous I or P frame with forward motion vector (MVfx,MVfy), and backward estimated from the following P frame with the backward motion vector (MVbx,MVby). The pixel value in B frame will be predicted by the respected pixels in previous frame and following frame with interpolation method.
### Motion Estimation
To estimate the macro-block in current frame ( P frame ) is moved from a macro-block on the previous frame ( I or P frame ) in the search area ( searching window ).
- Searching window : the area cover ± 7 (± 15) pixels centroid on the place of current macro-block.
- Motion Estimation : to find the best match M-block which embedded minimal MSE with the current M-block.
- Motion Vector : the distance between best match M-blocks and current M-blocks. MV( ± 7, ± 7 ) or MV( ± 15, ± 15 ).
- Motion Compensation : the difference between the current M-block and the best match M-block are coded by JPEG for compensating in the decoding process.

## MPEG-2[^4]
[^4]:[MPEG-2原理](http://vaplab.ee.ncu.edu.tw/~cfwang/4.html)
- A/V broadcast (TV, HDTV, Terrestrial, Cable, Satellite, High Speed Inter/Intranet) as well as DVD video
- MPEG-2 Audio
    - Support 5.1 channel
    - MPEG2 AAC: requires 30% fewer bits than MPEG1 layer 3
- Different DCT modes and scanning methods are developed for interlaced sequences.
![](https://i.imgur.com/FyAepln.png)
- Data partition
    - All headers, MVs, first few DCT coefficients in the base layer
    - Can be implemented at the bit stream level
    - Simple
- SNR scalability
    - Base layer includes coarsely quantized DCT coefficients
    - Enhancement layer further quantizes the base layer quantization error
- Temporal scalability
- Spatial scalability
![](https://i.imgur.com/p0HkLal.png)
MPEG-2 encodes video into double layer:Base layer and Enhancement layer. Base layer has higher transmission priority than enhancement layer, when receiver receive two layer's data, base layer can be encoded individually to provide basic video quality.Enhancement layer provides some important data to combine with base layer to provide better video quality, so when some loss occurs to enhancement layer will not affect the video quality too much.
## MPEG-4
- Functionalities beyond MPEG-1/2
    - Interaction with individual objects
- The displayed scene can be composed by the receiver from coded objects
    - Scalability of contents
    - Error resilience
    - Coding of both natural and synthetic audio and video

![](https://i.imgur.com/SIhhcTz.png)
### Object-Based Coding
- Entire scene is decomposed into multiple objects
- Each object is specified by its shape, motion, and texture (color)
    - Shape and texture both changes in time (specified by motion)
- MPEG-4 assumes the encoder has a segmentation map available, specifies how to code (actually decode!) shape, motion and texture
![](https://i.imgur.com/1I8x1Xq.png)

![](https://i.imgur.com/GDP11hy.png)

## H264
### Applications
- It is aimed at very low bit rate, real-time
- low end-to-end delay, and mobile applications such as conversational services and Internet video
- Enhanced visual quality at very low bit rates andparticularly at rate below 24kb/s
### Common Technical Elements with other Standards
- 16x16 macroblocks
- Conventional 4:2:0 sampling of chrominance and association of luminance and chrominance data (note 4:2:2 and 4:4:4 being planned for future revision)
- Block motion displacement
- Motion vectors over picture boundaries
- Variable block-size motion compensation
- Block transforms
- Scalar quantization
- I, P and B picture types
### New Features of H.264
* Variable block-sized motion compensation with small block size
* Multi-mode, multi-reference motion compensated
* Motion vector can point out of image border
* 1/4-, 1/8-pixel motion vector precision
* B-frame prediction weighting
* 4x4 integer DCT transform
* Multi-mode intra-prediction
* In-loop de-blocking filter
* UVLC (Uniform Variable Length Coding)
* SP-slices


## Channel coding
### Linear Block code
Assume the original data has k bits, plus n-k bits of parity check bits, the Code Rate R = k/n, when the data length and parity check bits length have linear relationship, it is called linear block code.
### cyclic code
指說一段長度為n bits的編碼，本身屬於finite field(GF field)，經過循環位移(如圖)後，仍然屬於GF field，這樣的結構性質可以運用在錯誤控制編碼。
![](https://i.imgur.com/KWWiag0.png)
### RS code
透過GF field來進行，以GF(8)為例p(x)=x^3+x+1 本質元素 滿足α^3+α+1=0 and a^7=1
GF(8)總共有8個元素，可以以指數和多項式型態來表示，運用於錯誤控制碼 過程太複雜，在此不敘述
![](https://i.imgur.com/IPb1FsX.png)
### CIRC(cross-interleaved Reed–Solomon code) coding
Codecs used by CD drives, including two Reed–Solomon Codes, (32,28) and (28,24) respectively, each symbol 8 bytes, the first one (32,28), so redundancy is 4 bits, which can correct two error, and so does (28,24), and use special technique called cross interleaving to combine them。32 bits And there is total (28/32)x(24/28)=75% contains data. The advantage is that there isn't too many errors it can be corrected directly.If there is too many errors it can detect and create repaired audio by interpolation method.
With errror control code, if there are small scratches, dust and fingerprints on CD and still in range of repair ability, then the audio sounds no differene.
## Watermarking
- Robustness：A digital watermark is called robust if it resists a designated class of transformations. Robust watermarks may be used in copy protection applications to carry copy and no access control information.
- Perceptibility：A digital watermark is called imperceptible if the original cover signal and the marked signal are perceptually indistinguishable
![](https://i.imgur.com/gIfwhui.png)
## Refs
[^5]
[^6]
[^5]:[Magnitude, real and phase images](http://vaplab.ee.ncu.edu.tw/~cfwang/4.html)
[^6]:[FFT of image data: “mirroring” to avoid boundary effects](https://dsp.stackexchange.com/a/1110)
