# poppler-subpixel
Subpixel glyphs rendering support to poppler and others

This repository contains some patches I made in order to make my evince
support subpixel rendering.

## Gallery

Left side: subpixel rendering result (after patch)

Right side: gray-scale rendering result (before patch)

Click to Enlarge:

![Journal paper](https://github.com/zhou13/poppler-subpixel/raw/master/img/journal.png)

![Liberation serif](https://github.com/zhou13/poppler-subpixel/raw/master/img/liberation_serif.png)

![CJK rendering](https://github.com/zhou13/poppler-subpixel/raw/master/img/cjk.png)

## cairo

The important patch is `cairo-lcdfilter-make-default-default.patch`.  This
patch switches the default lcdfilter of FreeType rendering backend of cairo
from `FT_LCD_FILTER_LEGACY` to `FT_LCD_FILTER_DEFAULT`.  This is necessary
because cairo does not provide a convenient way to specify the lcdfilter other
than fontconfig, which is not suitable for poppler.  In my opinion, this patch
should be merged into upstream as `FT_LCD_FILTER_LEGACY` _is only provided for
comparison purposes, and might be disabled or stay unsupported in the future_,
according to the document of FreeType.

Without patching cairo, the subpixel rendering still works but you may notice
more severe color fringes in glyphs due to the bad performance of
`FT_LCD_FILTER_LEGACY`.

## poppler

The patch `poppler-subpixel.patch` provides a new function called
`poppler_page_support_subpixel_rendering` for glib/cairo backend for poppler.
This function checks whether a PDF page could be subpixel-rendered.  The reason
that subpixel rendering is not implemented in may PDF viewer is that glyphs
subpixel rendering cannot easily support transparent background, i.e., the
background color much be known when the glyphs are rendered.  By default, PDF
has a transparent background.  In most case, one can fill white to the
background before the rendering of glyphs to workaround this problem. However,
PDF file format has a feature called [_blend
modes_](https://en.wikipedia.org/wiki/Blend_modes).  Example can be downloaded
from [https://github.com/mozilla/pdf.js/blob/master/test/pdfs/blendmode.pdf].
Some blend modes will generate different results when drawing on transparent or
white background.  Therefore, filling color into background is incorrect in
this case.  The new added function `poppler_page_support_subpixel_rendering`
checks whether it is safe to fill some color into background by traversing
current page and check whether transparent background and colored background
are distinguishable by all the blend modes used before rendering.  If it
returns true, the application can call `cairo_paint` to fill the background and
`cairo_font_options_set_antialias` to enable the subpixel rendering for cairo.

See `glib/demo/render.c` for an example how to use the subpixel rendering
feature.

## evince
The patch of evince uses `poppler_page_support_subpixel_rendering` to check
whether current page supports subpixel rendering.  If it is, fill white
background to the cairo surface and enable subpixel glyphs rendering in the
cairo engine.  Otherwise, insert white background afterward using
`CAIRO_OPERATOR_DEST_OVER` to guarantee the correctness.
