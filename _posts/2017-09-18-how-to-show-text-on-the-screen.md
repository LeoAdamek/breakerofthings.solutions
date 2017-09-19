---
layout: post
title: How to show text on the screen
date: 2017-09-18
---

# Introduction

This post was inspired by [@alex](https://github.com/alex)'s post [What happens when](https://github.com/alex/what-happens-when).
In this post I will attempt to explain in as much detail as I can, how you display text on the screen.
We're not going to take the easy option of using `printf`, which would simply hand the job directly to the operating system and we forget about it.
We want to render our own text on screen, and we're going to make it more interesting by rendering text that's a little less straightforward.

# The Result

The text we're going to render on screen is: **नमस्ते** — For those wondering that's "namaste", Hindi for "Hello."
A sample program can be found [here](https://github.com/LeoAdamek/how-to-show-text-on-screen/blob/master/how-to-show-text-on-screen.c)

Here's the end result we're looking for:

![Sample Output](/assets/text-shown-on-the-screen.png)

How did we get to this point? It seems simple enough, we just need to push these characters to screen right?
Not so fast. In the process of displaying this text on the screen, several things need to happen, all of which need to work in order to get the correct result.

1. We need to load our text into memory, with the correct encoding.
2. We need to load a typeface into memory, so we can map the text to glyphs from the font.
3. We need to load metadata about the characters we're rendering, so we can check for any special rules for rendering them.
4. We need to create a space where we can put the text.
5. We need to create an image of the text, correctly placing the glyphs from our typeface onto the rendering space.
6. We need to put the image on screen.

These are the six basic steps required to get our text on screen.


# The Tools

While we won't be writing _everything_ from scratch, and will use a few libraries to help us,
but there will be explanations about what each piece of code we use is doing, and how it gets us to our goal of showing text on screen.


The tools we'll be using are:

* C for the programming language.
* ICU for handling the unicode text and character information
* Freetype 2 for the font loading/rendering.
* Harfbuzz for laying out the text
* Cairo for rendering 2D graphics
* SDL for drawing the rendered text on screen.

I'm not going to cover setting up the C environment or libraries within this post, you can refer to the [source code][sauce] if you'd like to see the Makefile etc.

# 1. Loading the Text into Memory.

First we need to load the text into memory. If we were dealing with ASCII text, where each character is a single byte and the string is simply a list of bytes terminated by a `NUL` character (`0x00`)
we'd just need to use `char *text = "Hello";`

However, in order to properly handle the Hindi text, we're going to need to use Unicode. In this example we'll be using the ICU library, which is the most common reference implementation of Unicode.
In ICU, we use the `UChar` type to represent characters, which corresponds to an `unsigned short`. Internally the ICU library uses the UTF-16LE encoding, where most characters can be represented as a single
16-bit short integer. While the Unicode codespace extends beyond `0xFFFF`, none of the characters in the space beyond `0xFFFF` are in common use. 
These characters are mostly obscure or obsolete characters for Asian scripts, such as obscure Chinese Han, or Japanese _Hentaigana_.
This area also contains the characters for historical scripts not used for any modern language, such as ancient Egyptian heiroglyphs, and Cunieaform.

First, we need to include the relevant headers from the ICU library:

~~~~c
#include <unicode/utypes.h>
~~~~

This header defines all the base types used in the ICU library, such as `UChar`. While on most systems `UChar` will be an alias for `unsigned short`, it may differ on some systems.
And now we can declare our string:

~~~~c
UChar *text = u"नमस्ते";
~~~~

# 2. Loading a Typeface into Memory.

We've got the encoded data for our text in memory now, if we were simply looking to send it to a text terminal, we could probably stop here, however in this case we actually want to have control
of how our text is rendered. We'll need to load a typeface so we have a way of mapping the characters encoded in memory, to glyphs, graphical shapes that are used to represent a character.

To do this we'll be using the FreeType 2 library, another ubiquitous C library used by many projects, including most operating systems.
First, as always we need to include some headers:

~~~~c
#include <ft2build.h>
#include FT_FREETYPE_H
#include <freetype/ftadvanc.h>
#include <freetype/ftsnames.h>
#include <freetype/tttables.h>
~~~~

The `ft2build.h` header includes a definition for `FT_FREETYPE_H` which is a list of other headers needed on the system being built on.
With FreeType, each application maintains its own "library", used to handle the connection between the program and the FreeType 2 libraries.

First we need to initialize our own library:

~~~~c
FT_Library lib;
assert(!FT_Init_Freetype(&lib));
~~~~

We're using `assert()` here, from `<assert.h>` as a simple way to halt execution should something fail. We'll use this pattern many times in our program to handle possible, but unlikely failures.

Once we have a `FT_Library` set up, we now need to load some fonts into that library. A typeface is represented with the `FT_Face` type. Any program which needs to use multiple fonts will need a way to maintain
a collection of multiple `FT_Face`s and be able to address them as needed. As we're rendering only one word in one script, in one font, we'll just declare a single `FT_Face`.
We need to load a font file from system storage, into that typeface, and associate it with the library.

~~~~c
FT_Face typeface;

if (!FT_New_Face(lib, "fonts/our-font.ttf", 0, &typeface)) {
    fprintf(stderr, "Unable to load font file.");
    exit(EXIT_FAILURE);
}
~~~~

In this case we've not used `assert()` to handle failure, as not finding a specific font file on the disk is relatively likely failure, unlike initializing FreeType, which is much less likely to fail.
We need to tell Freetype how big this typeface should be, as Freetype is built for typography, it uses the common typographical unit for size, the point, defined as 1/72 inches. Because we're displaying
this text on a graphical display, we need to get the size in pixels, in the case of Freetype 2, we need to provide both the font size in points, as the pixel density of the display, both vertical and horizontal,
so that Freetype can convert the point values to pixel ones.

~~~~c
assert(!FT_Set_Char_Size(typeface, 0, 72, 90, 90));
~~~~

In the above code we set the size to 72pt, and the display to having 90 pixels per inch, both horizontally and vertically.

# 3. Loading Metadata for Characters to Check for Special Rendering Rules

Our particular text comes with some challenges. While in most European languages, we simply have a set of characters which get placed in order, left-to-right
and do not change, with the optional exception of ligatures. The text we're rendering is in the Devanagari script, an Indic script originally used for writing Sanskrit. 
Most Indian languages now use Devanagari or a derivative of it. The Devanagari script doesn't follow the same rules as European scripts, instead of characters being written
left-to-right in order, there are various combinations of characters which will change the order they are written, and their placement some letters even have multiple glyphs, 
and the surrounding text is used to determine the exact glyph which should be used. Because of this, the order of the characters as rendered, 
will not always match the order that they are stored in the encoded source text.

A full explanation, and source for much of the information in this post, can be found in [Chapter 12.1 of the Unicode 10.0 specification][unicode-spec-ch12]

We're going to simplify this part by using [HarfBuzz][harfbuzz] to tell us how to lay out the text. How the text lays out depends both on the characters themselves, 
and the font being used to display them.

First we tell Harfbuzz about the font we want to use:

~~~~c
hb_font_t hb_font = hb_ft_font_create(font, NULL);
hb_face_t hb_face = hb_ft_face_create(font, NULL);
~~~~

Next we also need to tell Harfbuzz what we're rendering, we first need to tell it some information about our text, such as the language and reading direction,
so that it knows which rules to apply. All of this data is stored in the `hb_buffer_t` type. In this case we're reading left-to-right, in Devanagari script, in Hindi.

~~~~c
hb_buffer_t buff = hb_buffer_create();
hb_buffer_set_direction(buff, HB_DIRECTION_LTR);
hb_buffer_set_script(hb_buff, HB_SCRIPT_DEVANAGARI);
hb_buffer_set_language(hb_buff, hb_language_from_string("hi", strlen("hi"));
~~~~

Now Harfbuzz knows about the text we want to render, we need to give it our actual text so it can lay it out.

~~~~c
hb_buffer_add_utf16(buff, text, u_strlen(text), 0, u_strlen(text));
~~~~

Finally, we can ask Harfbuzz to calculate which glyphs from the font we need to use, and where they need to be, relative to each other.
We then need to extract that information for later use.

~~~~c
unsigned int glyph_count;                                                   /* Will be used to store the total number of glyphs used in the text */
hb_shape(hb_font, buff, NULL, 0);                                           /* Calculate the glyphs and their positions */
hb_glyph_info_t *glyph_info = hb_buffer_get_glyph_info(buff, &glyph_count); /* Get information such as which glyphs to use */
hb_glyph_position_t *glyph_positions = hb_buffer_get_glyph_positions(buff, &glyph_count) /* Get the positions of each glyph */
~~~~


# 4. Creating a Space to Render the Text.

We'll be using [Cairo][cairo] to handle the actual rendering, and using [SDL][sdl] for interfacing with the display.

First we have to initialize SDL so we have the system's video capabilities available to our program. 
We also need create an area on the screen where our end result will appear.

~~~~c
#define VIDEO_WIDTH 800
#define VIDEO_HEIGHT 600
#define VIDEO_FLAGS SDL_SWSURFACE | SDL_RESIZABLE | SDL_DOUBLEBUF

asset( SDL_Init(SDL_INIT_VIDEO) >= 0);
SDL_WM_SetCaption("Some text on the screen", "Some text on the screen");


SDL_Surface *screen = SDL_SetVideoMode(WIDTH, HEIGHT, 32, VIDEO_FLAGS);

SDL_EnableUNICODE(1);
~~~~

This code initializes SDL so we have access to video functions, and allocates an area of 800×600 pixels, with 32-bit colour depth, 
rendered in software (using the main CPU, not using a dedicated graphics processor), and double-buffered.

Now we have this, we need to allocate some memory so we have a place to store the graphics we generate

~~~~c
SDL_Surface *render_area = SDL_CreateRGBSurface(VIDEO_FLAGS, VIDEO_WIDTH, VIDEO_HEIGHT, 32, 0x00ff0000, 0x0000ff00, 0x000000ff, 0x00000000);
~~~~

We also need to tell the cairo library about this memory area so it knows where to place its generated graphics.
Finally, we need to create a cairo resource, which is what we'll pass to the various drawing functions to create the graphics.
~~~~c
cairo_surface_t *cairo_surface = cairo_image_surface_create_for_data(
    (unsigned char*)render_area->pixels,
    render_area->w,
    render_area->h,
    render_area->pitch
);

cairo_t* cr = cairo_create(cairo_surface);
~~~~


# 5. Rendering the text onto the image.

At this point we now have the following in preparation for showing our text:

* The font we want to use to display the text
* Information about with glyphs from the font we need to use, in which order to create that text
* Information about where each glyph will need to go so the text reads correctly
* A place to put our generated output ready for display on the screen
* A connection to the computer's graphics capabilities so we can output graphics to the screen.

Now, we're ready to generate the graphical representation of our text.

First, we need to tell cairo about the font we want to use, as well as pass on the data Harfbuzz gave us about how to show it.

~~~~c
cairo_font_t *cairo_font = cairo_ft_font_face_create_for_ft_face(font, 0);

/* Create an array of cairo_glyph_t long enough to hold all the glyphs */
cairo_glyphs_t *cairo_glyphs = malloc(glyph_count * sizeof(cairo_glyph_t));

/* Initial pixel position within image */
unsigned int x = 100;
unsigned int y = 100;

for ( int i = 0; i < glyph_count; ++i ) {
    cairo_glyphs[i].index = glyph_info[i].codepoint;     /* Tell cairo which glyph to use */
    cairo_glyphs[i].x = x + glyph_positions[i].x_offset; /* Tell cairo the absolute position of the glyph*/
    cairo_glyphs[i].y = y - glyph_positions[i].y_offset;

    /* Move the pixel position for the next character */
    x += glyph_positions[i].x_advance - 20;
    y -= glyph_positions[i].y_advance;
}
~~~~

Now we've told cairo about the glyphs, we don't need Harfbuzz’s version of the data any more, so let’s free that memory:

~~~~c
free(glyph_info);
free(glyph_positions);
~~~~

Now we can generate the graphics.

~~~c
SDL_FillRect(render, NULL, 0xffffffff); /* Fill the background as white */

/* Set the current font face, size and colour */
cairo_set_font_face(cr, cairo_font);
cairo_set_font_size(cr, PT_SIZE);
cairo_set_source_rgba(cr, 0, 0, 0, 1); /* Solid black */

/* Draw the glyphs onto the surface */
cairo_show_glyphs(cr, cairo_glyphs, glyph_count);
~~~

We've now generated all the graphics data so that we can display the text, so we can move on to the final step, showing this data on screen.

# 6. Showing the image on screen.

We've got our graphics ready, we just need to show it on the screen now:

~~~~c
/* This will show the created graphics 
SDL_BlitSurface(render_area, NULL, screen, NULL);

/* We "flip" the screen, so that if we have more graphics to show, we can keep the current image on screen until we're ready to display them */
SDL_Flip(screen);
~~~~

At this point, we should see the result from the start of the post on screen.
After this we can repeat the process to show more text, or if we're done, clean up any resources we don't need any longer, and then wait for the user to close the window.

# 7. Summary

In conclusion, the process a modern computer goes through in order to show text on the screen has lots of hidden complexity, and relies on the work of hundreds of skilled programmers
in order for it to work successfully. This process, or a similar one with most of the same steps is required every time text needs to appear on screen.

If you have any suggestions for improvements to this post, please message _how-to-show-text-on-screen-suggestions [AT] breakerofthings [DOT] tech_

[sauce]: https://github.com/LeoAdamek/how-to-show-text-on-screen
[cairo]: https://www.cairographics.org/
[sdl]: https://www.libsdl.org
[harfbuzz]: https://www.freedesktop.org/wiki/Software/HarfBuzz/
[unicode-spec-ch12]: http://www.unicode.org/versions/Unicode10.0.0/ch12.pdf
