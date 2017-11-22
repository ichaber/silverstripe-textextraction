# Text extraction module

[![Build Status](https://secure.travis-ci.org/silverstripe-labs/silverstripe-textextraction.png)](http://travis-ci.org/silverstripe-labs/silverstripe-textextraction)
[![Code Quality](http://img.shields.io/scrutinizer/g/silverstripe-labs/silverstripe-textextraction.svg?style=flat-square)](https://scrutinizer-ci.com/g/silverstripe-labs/silverstripe-textextraction)
[![Version](http://img.shields.io/packagist/v/silverstripe/textextraction.svg?style=flat-square)](https://packagist.org/packages/silverstripe/silverstripe-textextraction)
[![License](http://img.shields.io/packagist/l/silverstripe/textextraction.svg?style=flat-square)](license.md)


Provides a text extraction API for file content, that can hook into different extractor
engines based on availability and the parsed file format. The output returned is always a string of the file content.

Via the `FileTextExtractable` extension, this logic can be used to
cache the extracted content on a `DataObject` subclass (usually `File`).

The module supports text extraction on the following file formats:

 * HTML (built-in)
 * PDF (with XPDF or Solr)
 * Microsoft Word, Excel, Powerpoint (Solr)
 * OpenOffice (Solr)
 * CSV (Solr)
 * RTF (Solr)
 * EPub (Solr)
 * Many others (Tika)

## Requirements

 * SilverStripe ^3.1
 * (optional) [XPDF](http://www.foolabs.com/xpdf/) (`pdftotext` utility)
 * (optional) [Apache Solr with ExtracingRequestHandler](http://wiki.apache.org/solr/ExtractingRequestHandler)
 * (optional) [Apache Tika](http://tika.apache.org/)

## Installation

```js
composer require silverstripe/textextraction
```

The module depends on the [Guzzle HTTP Library](http://guzzlephp.org),
which is automatically checked out by composer. Alternatively, install Guzzle
through PEAR and ensure its in your `include_path`.

## Configuration

### Basic

By default, only extraction from HTML documents is supported.
No configuration is required for that, unless you want to make
the content available through your `DataObject` subclass.
In this case, add the following to `mysite/_config/config.yml`:

```yaml
File:
  extensions:
    - FileTextExtractable
```

By default any extracted content will be cached against the database row.
In order to stay within common size constraints for SQL queries required in this operation,
the cache sets a maximum character length after which content gets truncated (default: 500000).
You can configure this value through `FileTextCache_Database.max_content_length` in your yaml configuration.


Alternatively, extracted content can be cached using SS_Cache to prevent excessive database growth.
In order to swap out the cache backend you can use the following yaml configuration.


```yaml
---
Name: mytextextraction
After: '#textextraction'
---
Injector:
  FileTextCache: FileTextCache_SSCache
FileTextCache_SSCache:
  lifetime: 3600 # Number of seconds to cache content for

```

### XPDF

PDFs require special handling, for example through the [XPDF](http://www.foolabs.com/xpdf/)
commandline utility. Follow their installation instructions, its presence will be automatically
detected for \*nix operating systems. You can optionally set the binary path (required for Windows) in `mysite/_config/config.yml`

```yml
PDFTextExtractor:
  binary_location: /my/path/pdftotext
```

### Apache Solr

Apache Solr is a fulltext search engine, an aspect which is often used
alongside this module. But more importantly for us, it has bindings to [Apache Tika](http://tika.apache.org/)
through the [ExtractingRequestHandler](http://wiki.apache.org/solr/ExtractingRequestHandler) interface.
This allows Solr to inspect the contents of various file formats, such as Office documents and PDF files.
The textextraction module retrieves the output of this service, rather than altering the index.
With the raw text output, you can decide to store it in a database column for fulltext search
in your database driver, or even pass it back to Solr as part of a full index update.

In order to use Solr, you need to configure a URL for it (in `mysite/_config/config.yml`):

```yml
SolrCellTextExtractor:
  base_url: 'http://localhost:8983/solr/update/extract'
```

Note that in case you're using multiple cores, you'll need to add the core name to the URL
(e.g. 'http://localhost:8983/solr/PageSolrIndex/update/extract').
The ["fulltext" module](https://github.com/silverstripe-labs/silverstripe-fulltextsearch)
uses multiple cores by default, and comes prepackaged with a Solr server.
Its a stripped-down version of Solr, follow the module README on how to add
Apache Tika text extraction capabilities.

You need to ensure that some indexable property on your object
returns the contents, either by directly accessing `FileTextExtractable->extractFileAsText()`,
or by writing your own method around `FileTextExtractor->getContent()` (see "Usage" below).
The property should be listed in your `SolrIndex` subclass, e.g. as follows:

```php
class MyDocument extends DataObject {
	static $db = array('Path' => 'Text');
	function getContent() {
		$extractor = FileTextExtractor::for_file($this->Path);
		return $extractor ? $extractor->getContent($this->Path) : null;
	}
}
class MySolrIndex extends SolrIndex {
	function init() {
		$this->addClass('MyDocument');
		$this->addStoredField('Content', 'HTMLText');
	}
}
```

Note: This isn't a terribly efficient way to process large amounts of files, since
each HTTP request is run synchronously.

### Tika

Support for Apache Tika (1.8 and above) is included. This can be run in one of two ways: Server or CLI.

See [the Apache Tika home page](http://tika.apache.org/1.8/index.html) for instructions on installing and
configuring this. Download the latest `tika-app` for running as a CLI script, or `tika-server` if you're planning
to have it running constantly in the background. Starting tika as a CLI script for every extraction request
is fairly slow, so we recommend running it as a server.

## Bugtracker

Bugs are tracked in the issues section of this repository. Before submitting an issue please read over
existing issues to ensure yours is unique.

If the issue does look like a new bug:

 - Create a new issue
 - Describe the steps required to reproduce your issue, and the expected outcome. Unit tests, screenshots
  and screencasts can help here.
 - Describe your environment as detailed as possible: SilverStripe version, Browser, PHP version,
 Operating System, any installed SilverStripe modules.

Please report security issues to security@silverstripe.org directly. Please don't file security issues in the bugtracker.

## Development and contribution
If you would like to make contributions to the module please ensure you raise a pull request and discuss
 with the module maintainers.
