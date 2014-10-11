---
layout: post
title: 'Understanding ViBE: Part I'
date: 2011-04-10 18:42:28.000000000 -07:00
status: published
type: post
---
Around a this time last year I came across an algorithm that does 
efficient background subtraction. However, the author provided a binary 
blob instead of source code and so most of the embedded community was 
left out of being able to use this algorithm. I thought it would be a 
good chance to show how to reverse engineer a binary blob back into 
usable, portable C code. 

The author provided an archive containing a license and three relevant 
files: a header file, a binary blob and some example code. The hashes 
of the archive are:

{% highlight text %}
MD5 f0514aaf802f7cfe08f3a09c8eea0107
RMD160 3a2d6b0ed672c0e569b210c5b22d849d2741d6a2
SHA 5290d7d15f2cd5685e8970b3b49965b6645421ed
{% endhighlight %}

The public API follows:

 * `vibeModel_t *libvibeModelNew();`

    This is the constructor that allocates and initializes a new ViBE 
    model for use.

 * `int32_t libvibeModelFree(vibeModel_t *model);`

    This is the destructor that complements the constructor above.

 * `uint32_t libvibeModelPrintParameters(const vibeModel_t *model);`

    As its name suggests, this function prints the parameters associated 
    with this algorithm's instance. It is likely that it will help describe 
    what members are in the vibeModel_t structure.

 * `uint32_t libvibeModelGetNumberOfSamples(const vibeModel_t *model);`
 * `uint32_t libvibeModelGetMatchingThreshold(const vibeModel_t *model);`
 * `uint32_t libvibeModelGetMatchingNumber(const vibeModel_t *model);`
 * `uint32_t libvibeModelGetUpdateFactor(const vibeModel_t *model);`

    These functions are getters for parameters used in the algorithm. 
    It's likely they will map directly to entities in the structure.

 * `int32_t libvibeModelSetNumberOfSamples(vibeModel_t *model, const uint32_t numberOfSamples);`
 * `int32_t libvibeModelSetMatchingThreshold(vibeModel_t *model, const uint32_t matchingThreshold);`
 * `int32_t libvibeModelSetMatchingNumber(vibeModel_t *model, const uint32_t matchingNumber);`
 * `int32_t libvibeModelSetUpdateFactor(vibeModel_t *model, const uint32_t updateFactor);`

    These are setters which compliment the above getters. 

 * `int32_t libvibeModelAllocInit_8u_C1R(vibeModel_t *model, const uint8_t *image_data, const uint32_t width, const uint32_t height, const uint32_t stride);`
 * `int32_t libvibeModelAllocInit_8u_C3R(vibeModel_t *model, const uint8_t *image_data, const uint32_t width, const uint32_t height, const uint32_t stride);`

    These functions allocate the memory required for specific kinds of 
    input.

 * `int32_t libvibeModelUpdate_8u_C1R(vibeModel_t *model, const uint8_t *image_data, uint8_t *segmentation_map);`
 * `int32_t libvibeModelUpdate_8u_C3R(vibeModel_t *model, const uint8_t *image_data, uint8_t *segmentation_map);`

    These functions update the stored image data and the instance's 
    segmentation map.

 * `int32_t libvibeModelFree(vibeModel_t *model);`

    This is the destructor that complements the constructor above.

From the header file given in the distribution, I can gather some 
information about what structure members are in the ViBE Model 
structure. From this API I gather that an instance's vibeModel_t is 
often called the identifier "model" and that most functions and 
variables are camel case, and with the getters and setters of the 
algorithm's parameters. This approach shows that the author decided to 
tackle the problem in a state-ful way using C structures as "objects". 
I am beginning to doubt that this algorithm is "under 100 lines of 
code".

I created a header file to hold all function prototypes, data 
structures, and defined constants. I added what exposed function 
prototypes in the documentation to this header file, and then began 
disassembling the code into its parts. From the header provided, I can 
gather some basic information about what the code looked like.

(Continued in [part II][1]).

  [1]: {{ site.baseurl }}/post/understanding-vibe-part-ii
