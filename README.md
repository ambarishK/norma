# Norma

_edited 20150326_
_edited 20150412_
_ediedt 201504016_

================================
NOTE: Norma does not do all transformations. The current position is:
- main branch does XML to scholarly HTML only
- petermr / dev fork/branch does PDF 2 TXT based on PDFBox.


The PDF2HTML code is in there but it's messy. I'd guess a week or two to get the logic working, but the results will be very variable depending on the type of PDF.
================================

A tool to convert a variety of inputs into normalized, tagged, XHTML (with embedded/linked SVG and PNG where
appropriate). The initial emphasis is on scholarly publications but much of the technology is general.  

## Overview

See https://github.com/petermr/norma .

## History

Originally AMI (http://bitbucket.org/petermr/ami-core) involved the conversion of legacy inputs (PDF, XML, HTML, etc.) into well-formed documents which were then searched by AMI-plugins. This required a Visitor pattern
with _n_ inputs and _m_ visitors. The inputs could be PDF, SVG, HTML, XML, etc. and AMI often had to convert. This became unmanageable.
AMI has therefore been refactored into `ami2`.

## Current architecture

We have now split this into two components:

  * norma. This takes legacy files (more later) and converts to normalized, tagged, XHTML (`scholarly.html`)
  * ami2. This reads `scholarly.html` and applies one or more plugins.
  
The workflow is then roughly

```
                   quickscrape              
                       ||
                       CM
                       ||
                       V
single files => {PDF, HTML, ...} => Norma => NHTML => AMI => results (XML or JSON)
                                      ^                ^
                                      |                |
                                    tagger           plugin
                    
```

## Input

There are two modes of input to `norma`:
 * single legacy files (e.g. `-i my/dir/foo.html`). A new CM directory is created from the `-o` argument, e.g `-o /some/where` creates a new CM directory `/some/where/`.  `foo.html` is then copied to `/some/where/fulltext.html` (`fulltext.html` is a reserved name triggered by the input `.html` suffix.)
 * pre-existing CM directories with one or more reserved files. These are normally generated by a `quickscrape` operation and the user does not need to copy or rename anything. Note that `quickscrape`-generated CMs should always have a `results.json` file, but some or all of the `fulltext.*` files may be missing since they didn't exist or weren't downloaded.
 


## Taggers and Normalization

 *Taggers are not yet standard*
 
Each document type has a tagger, based on its format, structure and vocabulary. For example an HTML file from
PLoSONE is not well-formed and must be converted, largely automatically. Some documents are "flat" and must be grouped into sections (e.g. Elsevier and Hindawi HTML have no structuring ``div``s. After normalization the tagger will identify sections
based on XPaths in pubstyle; and add tags which should be standardized across a wide range of input sources.

### Format

Taggers will be created in XML, and use XSLT2 where possible. The tagger carries out the following:

 * normalize to well-formed XHTML. This may be automatic but may also require some specific editing for some of the
 worst inputs (lack of quotes, etc.) We use Jsoup and HtmlUnit as appropriate and these do a good job most of the time.
 * structure the XHTML. Some publications are "flat" (e.g h2 and h3 are siblings) and need structuring (XSLT2)
 * tag the XHTML. We use XPath (with XSLT2) to add ``tag`` attributes to sections. These can be then recognized by AMI using the ``-x`` xpath-like option.
 
### Development of taggers

In general each publication type requires a tagger. This is made much easier where a publisher uses the same toolchain for all its publications  (e.g. BMC, Hindawi or Elsevier). We believe that taggers can be developed by most people, given a supportive community. The main requirements are:

 * understand the structure and format of the input.
 * work out how this maps onto similar, solved, publications.
 * write XPath expressions for the tagging. If you understand filesystems and the commandline you shouldn't have much problem and we believe it should take about 15-30 minutes.
 * use established tag names for the sections.
 
## Transformation and Legacy formats

Most input files will need transformation. "html" files are often not W3C-compliant and in any case need normalization and tagging. All  `xml` and `pdf` files need significant transformation. In most cases the default output format is `scholarly.html`. In some cases the transformation mechanism must be made specific (e.g. to transform `xml` to `html` requires an XSL stylesheet and the argument `--xsl foo.xsl` will trigger the transformation of the `-i` input file to the `-o` output file.

### HTML

There are several normalization processes and often all are needed:
 * _encoding_. Some HTML files do not use UTF-8 and it may be neceesary to convert the output encoding. Ones from MS tools are often a problem. 
 * _character normalization to Unicode_. This applies to all input formats and we try to convert where possible.
 * _well-formedness_. HTML is not required to be compliant `XML` (`XHTML`) and it is necessary to make it so. This is not guarantted to be possible - some HTML is so awful that it is impossible to parse. HTML5 need not be well formed, but if compliant it must conform to the W3C spec. There are a number of tools that try their best (Jtidy, JSoup, Tagsoup, HtmlUnit). Most will output well-formed HTML but some will refuse with some documents. We therefore offer a default choice based on experience but allow the user to choose others (experimental).
 * _structuring_. Many HTML documents are "flat" (e.g. Hindawi press), i.e. there are no container elements such as `div`. We thne have to try to group all the elements in a section together. This may be done by an `XSLT2` spreadsheet or we may have to use custom code.
 

### PDF

The main legacy format is PDF. We provide two Java-based converters (PDF2TXT, and PDF2XML). PDF2TXT can be called automatically from Norma with the `--xsl pdf2txt` flag. PDF2XML is more sophisticated and deals with graphics and layout. 

All PDF converters are lossy and likely to fail slightly or seriously on boxes, tables, etc. PDF2XML concentrates on science and particularly mathematical symbols, styling/italics and sub/superscripts. You may have other converters which do a better job of some of the parts - and should configure them to output HTML (which we can normalize later.

PDF2TXT creates a single file `fulltext.pdf.txt`. This loses all formatting and style and  often has words or blocks in the wrong order. That's not PDFBox's fault, it's the fundamental problem of PDF.

PDF2XML convert to a single file, but in most cases it will be a collection. So there will be a special subdirectory `pdf`.

### SVG

Some graphics is published as vectors within PDF and these can be converted into SVG. The SVG is wrapped in NHTML and can, in certain cases such as chemistry, be converted into science. It's possible that formats such as EPS or WMF/EMF can be converted into SVG.

Where the `SVG` is created by processing `PDF` it will be in a subdirectory of `pdf`. 

### DOC/X

Some years ago I wrote a semi-complete parser for DOCX but it's lost... Probably could resurrect if required. Most likely use would be theses or possibly arXiv. Output would probably be `html` and `svg` in a `docx` subdirectory.

### LaTeX

No specific tools yet. I'd like to know if there is a parsable intermediate output from LaTeX (e.g. before the `*.dvi`). Otherwise we have to recreate a LaTeX parser which I'd prefer not to do.

## Output

The preferred output method is to offer a QuickscrapeNorma (CM) (`-q`) option, to which output files can be added. A CM is essentially a directory 
with conventional names for files and subdirectories. The minimal output is likely to be a single `scholarly.html` 
 The `scholarly.html` should always be well-formed, Unicode, structured XHTML. That makes it a significant output in its 
own right.

Thus 
```
norma -q foo/bar/
```
will expect a variety of files in ```foo/bar``` created either by `quickscrape` or repeated applications of other CM software (e.g. `norma`
or `ami2`). Norma will then output the normalised result as 
```
scholarly.html
```

Where Norma is not given a FileContainer then the output directory or file must be specified.

## Commandline arguments

see ARGS.md

The current arguments  are:

```
-i or --input  file(s)_and/or_url(s)
```
			
			Input stream (Files, directories, URLs), 
```
-q or --quickscrapeNorma  director(ies)
```
			
			create or use CM directory 

```
-o or --output  file_or_directory
```

			Output is to local filestore 
		
```
-r or --recursive 
```

			recurse through input directories
		
```
-e or --extensions  ext1 [ext2...]
```

			input file extensions (may be obsolete)
```
-h or --help 
```

				outputs help for all options, including superclass DefaultArgProcessor
		
```
-c or --chars  [pairs of characters, as unicode points; i.e. char00, char01, char10, char11, char20, char21
```

			Replaces one character by another. (Not tested) 
		
```
-d or --divs  expression [expression] ...
```
			
			List of expressions identifying XML element to wrap as divs/sections
		
```
-n or --name   tag1,tag2 [tag1,tag2 ....]
```
			
			List of comma-separated pairs of HTML tags to change
		
```
-p or --pubstyle  pub_code
```

			Code or mnemomic to identifier the publisher or journal style. 

```
-z or --standalone  boolean
```
			
			Treats XML document as standalone. 
		
```
-s or --strip  [options to strip]
```
			List of XML components to strip from the raw well-formed HTML (not yet tested)

		
```
-t or --tidy  [HTML tidy option]
```
			
			Choose an HTML tidy tool. 
		
```
-x or --xsl  stylesheet
```
			
			Transform XML or HTML input with stylesheet. 


