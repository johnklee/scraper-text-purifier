[![Build Status](https://travis.ibm.com/Neuromancer/w4cs-text-purifier.svg?token=JKd6mMpdi1KyTrmdk8St&branch=develop)](https://travis.ibm.com/Neuromancer/w4cs-text-purifier)


# w4cs-text-purifier
Text extraction Python package for W4CS Crawler 2.0. The package requires Python 3.6 (or above) and
Java 1.7+ (for PDF extraction) to run.
* [**Usage**](#usage)
    * [API Introduction](#api_usage)
    * [Register customized strategy for text extraction](#api_reg_strategy)
    * [Extract Links from HTML](#api_extract_links)
    * [Register Post Processor](#api_regr_pp)
* [**Build, Test, and Distribute**](#build_test_dist)
* [**Directory Structure**](#directory_struct)

## Usage  <a name='usage'/>
### API Introduction  <a name='api_usage'>
From this package, class **TEAgent** will support below MIME in default for text extraction:

MIME type    | Handler
------------ | -------------
 `text/html` | **HTML2TextHandler** ([readability-lxml](https://pypi.org/project/readability-lxml/))
 `application/pdf` | **PDFTika2TextHandler** ([tika-python](https://github.com/chrismattmann/tika-python))
 `text/plain` | **PlainTextHandler**

Below demonstrate the basic usage of class **TEAgent**:
```python
from purifier import TEAgent  #  Text Extractor Agent

extractor = TEAgent()

is_succ, rst, handler = extractor.parse("text/html", "http://someurl", "<html><title>some title</title>hello world</html>")

print("extraction was successful? ", is_succ)
print("extraction was handled by: ", handler)
print("result text: \n", rst['text'])
```

Executing the code shows:
```console
extraction was successful?  True
extraction was handled by:  {'reason': ('HTML2TextHandler', None)}
result text:
 some title
hello world
```

### Register customized strategy for text extraction <a name='api_reg_strategy'/>
You can register your own text extraction strategy by below sample code:
```python
from bs4 import BeautifulSoup
from purifier import TEAgent  #  Text Extractor Agent

mime = "text/html"
url = "http://someurl"
html = "<html><title>some title</title>hello world</html>"
extractor = TEAgent()
my_te_result = "My extraction result"

# Define your own text extraction strategy
class MyStrategy:
    def __call__(self, url, content):
        r'''
        Implementation of customized text extraction to retrieve title only
        '''
        soup = BeautifulSoup(content, "html.parser")
        return soup.title.string

extractor.handlers[mime].regr(r'r:http://john.com/\d+', MyStrategy())


# Default handler
is_succ, rst, handler = extractor.parse(mime, url, html)
assert is_succ, 'Fail in parsing'
assert rst['text'].strip() == 'some title\n\nhello world'
assert handler == {'reason': ('HTML2TextHandler', None)}

# Customized handler
is_succ, rst, handler = extractor.parse(mime, 'http://john.com/123', html)
print(handler)  # {'reason': ('<__main__.MyStrategy object at 0x7f4d78077ef0>', None)}
assert is_succ, 'Fail in parsing'
assert rst['text'].strip() == 'some title'
assert 'MyStrategy' in str(handler['reason'][0])
```

### Extract Links from HTML <a name='api_extract_links'/>
class **TEAgent** also supoort you to extract link(s) from HTML. Check the sample code below:
```python
from purifier.html2text import *

url = 'https://john.com/2014/02/21/an-in-depth-analysis-of-linuxebury/'
h2t_handler = HTML2TextHandler()
html = r'''<html><title>For testing</title>
<h1>This is the head line</h1>
This is a link with text with <a href='http://absolute.com'>absolute path</a>.<br/>
This is a embedded URL in text as http://test.url.com<br/>
This is a <a href='/relative/index.html'>relative path URL</a>.<br/>
</html>
'''

text = h2t_handler.handle(url, html)
all_urls, bdy_urls = h2t_handler.extract_link_from_html(url, html, text)
bdy_txt_urls = h2t_handler.extract_link_from_text(text)



print('===== all_urls ({:,d}) ====='.format(len(all_urls)))
for u in sorted(all_urls):
    print(u)

print()

print('===== bdy_urls ({:,d}) ====='.format(len(bdy_urls)))
for u in sorted(bdy_urls):
    print(u)

print()

print('===== bdy_txt_urls ({:,d}) ====='.format(len(bdy_txt_urls)))
for u in sorted(bdy_txt_urls):
    print(u)

print()
```
Output will look like:
```console
===== all_urls (2) =====
http://absolute.com
https://john.com/relative/index.html

===== bdy_urls (2) =====
http://absolute.com
https://john.com/relative/index.html

===== bdy_txt_urls (1) =====
http://test.url.com
```
However, it is recommended to use class `TEAgent` to carry out the TE action. Below is the usage:
```console
>>> from purifier.text_extractor import *
>>> tea = TEAgent()
>>> html = r'''<html><title>For testing</title>
... <h1>This is the head line</h1>
... This is a link with text with <a href='http://absolute.com'>absolute path</a>.<br/>
... This is a embedded URL in text as http://test.url.com<br/>
... This is a <a href='/relative/index.html'>relative path URL</a>.<br/>
... </html>
...

# Extract text without retriving the link(s)
>>> is_suc, rst, reason = tea.parse('text/html', 'https://john.com/2014/02/21/an-in-depth-analysis-of-linuxebury/', html)
>>> rst
{'te_suc': True, 'text': 'For testing\nThis is the head line\n\nThis is a link with text with\nabsolute path\n.\n\nThis is a embedded URL in text as http://test.url.com\n\nThis is a\nrelative path URL\n.', 'title': ''}

# Extract text and retrieve link(s) by giving argument do_ext_link=True
>>> is_suc, rst, reason = tea.parse('text/html', 'https://john.com/2014/02/21/an-in-depth-analysis-of-linuxebury/', html, do_ext_link=True)
>>> from pprint import pprint
>>> pprint(rst)
{'all_links': ['https://john.com/relative/index.html', 'http://absolute.com'],
 'body_a_links': ['https://john.com/relative/index.html',
                  'http://absolute.com'],
 'body_txt_links': ['http://test.url.com'],
 'te_suc': True,
 'text': 'For testing\n'
         'This is the head line\n'
         '\n'
         'This is a link with text with\n'
         'absolute path\n'
         '.\n'
         '\n'
         'This is a embedded URL in text as http://test.url.com\n'
         '\n'
         'This is a\n'
         'relative path URL\n'
         '.',
 'title': ''}
```
### Register Post Processor <a name='api_regr_pp'/>
You can define your own post process to retrieve more information after text extraction such as title, publish date of article
Currently, the post process to title extraction is an built-in option and you can turn it on by below sample cdoe:
```python
>>> from purifier.text_extractor import *
>>> tea = TEAgent(ext_title=True)  # Turn on post processor to extract title
>>> import requests
>>> url = 'https://www.ibm.com/tw-zh/'
>>> r = requests.get(url)
>>> mime = r.headers.get('content-type')
>>> html = r.text
>>> is_suc, rst_dict, reason = tea.parse(mime, url, html)
>>> is_suc
True
>>> rst_dict['title']       # Obtain the title of page
'IBM 台灣官方網站丨全球領先的人工智慧解決方案與雲平台公司 - 台灣'
>>> rst_dict['text'][:30]   # Obtain the text extraction result
'IBM 台灣官方網站丨全球領先的人工智慧解決方案與雲平台公司'
>>> tea.handlers['text/html'].pp_list   # The post processor to extract title
[<purifier.html2text.P4Title object at 0x7f4eb0428ef0>]
```

## Build, Test, and Distribute <a name='build_test_dist'/>
```bash
$ make          # unit tests and linting check
$ make metrics  # test if text extraction drops
$ make dist     # build package to dist/purifier-#.#.#.tar.gz
```

## Directory Structure  <a name='directory_struct'/>

```
.
├── Makefile           # install, build, test targets
├── README.md          # this file
├── metrics            # contains data and scrpts for generating metrics
├── purifier           # main package
├── requirements.txt   # development dependent 
├── setup.py           # package setup
├── tests              # unit tests
└── tools              # utility used internally to facilitate development
```
