---
description: In this tutorial, we will find some broken links to section headers.
tags:
  - python
  - gitbook
---

# Finding broken links to section headers

When I was on team that used GitBook to author and publish documentation, I found it was difficult to know if updating a section header was going to break an existing link to that header., so I wrote a python script to find broken links to headers.

## Prerequisites

* Install [Python](https://www.python.org/).
* Download the [header-links-tutorial](https://github.com/mikeromard/public-docs/tree/header-links-tutorial) branch of my fork of GitBook's [public-docs](https://github.com/GitbookIO/public-docs) repo.

## The script

{% code title="check_header_links.py" lineNumbers="true" %}
```python
import os
import logging
import re

root_folder = os.path.expanduser("~/Projects/public-docs/") # this is where I have my fork of GitBook's public-docs repo downloaded
path_to_summary = os.path.expanduser(f"{root_folder}SUMMARY.md") # this is the path to the SUMMARY.md file
md_link_pattern = re.compile(r"\[[\S ]*\]\(([\S]*.md)\)")
md_header_link_pattern = re.compile(r"\[[\S ]*\]\(([\S]*.md)#([\S]*)\)")

def get_md_page_paths(path_to_summary):
    with open(path_to_summary, "r") as summary_file:
        md_page_paths = re.findall(md_link_pattern, summary_file.read())
        return md_page_paths
        
def get_header_links_in_pages(page):
    with open(page, "r") as source:
        header_links = re.findall(md_header_link_pattern, source.read())
        return header_links

def main():
    logging.basicConfig(filename="check_header_links.log", level=logging.ERROR, format="{asctime} - {levelname} - {message}", style="{", datefmt="%Y-%b-%d %H:%M:%S", filemode="w")
    logging.info("Checking for broken header links.\n")
    source_files = get_md_page_paths(path_to_summary)
    for file in source_files:
        logging.info(f"Checking {file}.\n")
        filepath = f"{root_folder}{file}" # this is the full path to the file, including the filename
        directory_name = os.path.dirname(filepath) #this is the full path to the file without the filename
        header_links = get_header_links_in_pages(filepath)
        if len(header_links) == 0:
            pass
        else:
            for header_link in header_links:
                target_source_file = header_link[0]
                target_anchor = f"{header_link[1]}"
                if target_anchor.startswith("id-"): # this is specific to GitBook's docs at the time of writing
                    target_anchor = target_anchor.replace("id-","")
                if "azure-a-d" in target_anchor: # this is specific to GitBook's docs at the time of writing
                    target_anchor = target_anchor.replace("azure-a-d", "azure-ad")
                target_href = f'href="{target_anchor}"'
                target_id = f'id="{target_anchor}"'
                target_anchor = target_anchor.replace('-', ' ')
                target_header = f"# {target_anchor}"
                target_summary = f"<summary>{target_anchor}"
                target_bold = f"**{target_anchor}"
                logging.debug(f"Target header: {target_header}")
                logging.debug(f"Target summary: {target_summary}")
                logging.debug(f"Target bold: {target_bold}")
                logging.debug(f"Target href: {target_href}")
                logging.debug(f"Target ID: {target_id}\n")
                target_source_file = os.path.normpath(os.path.join(directory_name, target_source_file))
                if os.path.isfile(target_source_file):
                    with open(target_source_file, "r") as target_file:
                        target_file_content = target_file.read()
                        if target_header.lower() in target_file_content.lower():
                            break
                        elif target_summary.lower() in target_file_content.lower():
                            break
                        elif target_bold.lower() in target_file_content.lower():
                            break
                        elif target_href.lower() in target_file_content.lower():
                            break
                        elif target_id.lower() in target_file_content.lower():
                            break
                        else:
                            logging.error(f"{filepath} contains a link to '{header_link[0]}#{header_link[1]}', but '{target_anchor}' not found in {target_source_file}\n")
                else: # this is to let us know if we've somehow got an invalid target_source_file
                    logging.error(f"{filepath} contains a link to {header_link}, but {target_source_file} isn't a file.\n")

if __name__ == '__main__':
    main()

```
{% endcode %}

### How the script works

The script extracts the filenames for entries in your `SUMMARY.md` file, opens each of those files, and checks for markdown links to other pages that include a `#` character, which indicates that the link goes to a section header or another anchor. When it finds one of these links, it opens the file being linked to and checks if that header or other anchor exists in that file. If it doesn't, it logs an error to a file called `check_header_links.log`.

Let's break it down into smaller pieces.

#### Importing modules

{% code title="" %}
```python
import os
import logging
import re
```
{% endcode %}

It starts by importing the [`os`](https://docs.python.org/3/library/os.html), [`logging`](https://docs.python.org/3/library/logging.html), and [`re`](https://docs.python.org/3/library/re.html) modules. We're using `os` to get some file path information, and to check to see if a file exists. We're using `logging` to output any errors we find, and we can also use that to help debug any issues that we run into. We're using `re` for regular expression (regex) patterns that we can use to find file paths and links.

#### Assigning variables

{% code title="" %}
```python
root_folder = os.path.expanduser("~/Projects/public-docs/") # this is where I have my fork of GitBook's public-docs repo downloaded
path_to_summary = os.path.expanduser(f"{root_folder}SUMMARY.md") # this is the path to the SUMMARY.md file
md_link_pattern = re.compile(r"\[[\S ]*\]\(([\S]*.md)\)")
md_header_link_pattern = re.compile(r"\[[\S ]*\]\(([\S]*.md)#([\S]*)\)")
```
{% endcode %}

Here we're assigning the local path to our repo as the `root_folder` variable, and the path within that folder to `SUMMARY.md` as the `path_to_summary` variable. We're using `os.path.expanduser` to transform the `~` into the full path. We're using a [formatted string literal](https://docs.python.org/3/tutorial/inputoutput.html#tut-f-strings) (f string) in `path_to_summary` so we can include the `root_folder` variable in it.

We're assigning regexes to `md_link_pattern`  and `md_header_link_pattern`.&#x20;

Explaining regex in depth is outside of the scope of this tutorial, but at a high-level:

* We're looking for any number (`*`) of non-whitespace characters (`\S`) and spaces ( ) within square brackets `\[[\S ]*\]`, followed by a markdown path in round brackets (`\(([\S]*.md)\)` or `\(([\S]*.md)#([\S]*)\)`).&#x20;
* We're using `\` to escape the square bracket and round bracket characters.
* We're using round brackets for capture groups. This is for the information we want to extract when we match the patterns. In `md_link_pattern` we're looking to get the file names called in `SUMMARY.md`, so we're capturing `([\S]*.md)`. In `md_header_link_pattern` we're looking for that too, but we're also capturing the non-whitespace characters that appear after the `#` character in the link `([\S]*)`.
* I find using a site like [https://regex101.com/](https://regex101.com/) is helpful when crafting a regex.&#x20;

#### Get markdown page paths

{% code title="" %}
```python
def get_md_page_paths(path_to_summary):
    with open(path_to_summary, "r") as summary_file:
        md_page_paths = re.findall(md_link_pattern, summary_file.read())
        return md_page_paths
```
{% endcode %}

This function is given the path to `SUMMARY.md`, reads that file, and finds and returns all  `md_link_pattern` matches in it.

#### Get header links

{% code title="" %}
```python
def get_header_links_in_pages(page):
    with open(page, "r") as source:
        header_links = re.findall(md_header_link_pattern, source.read())
        return header_links
```
{% endcode %}

This function is given the path to a page, reads that file, and finds and returns all `md_header_link_pattern` matches in it.

#### The main() function

{% code title="" %}
```
def main():
    logging.basicConfig(filename="check_header_links.log", level=logging.ERROR, format="{asctime} - {levelname} - {message}", style="{", datefmt="%Y-%b-%d %H:%M:%S", filemode="w")
    logging.info("Checking for broken header links.\n")
    source_files = get_md_page_paths(path_to_summary)
    for file in source_files:
        logging.info(f"Checking {file}.\n")
        filepath = f"{root_folder}{file}" # this is the full path to the file, including the filename
        directory_name = os.path.dirname(filepath) #this is the full path to the file without the filename
        header_links = get_header_links_in_pages(filepath)
        if len(header_links) == 0:
            pass
        else:
            for header_link in header_links:
                target_source_file = header_link[0]
                target_anchor = f"{header_link[1]}"
                if target_anchor.startswith("id-"): # this is specific to GitBook's docs at the time of writing
                    target_anchor = target_anchor.replace("id-","")
                if "azure-a-d" in target_anchor: # this is specific to GitBook's docs at the time of writing
                    target_anchor = target_anchor.replace("azure-a-d", "azure-ad")
                target_href = f'href="{target_anchor}"'
                target_id = f'id="{target_anchor}"'
                target_anchor = target_anchor.replace('-', ' ')
                target_header = f"# {target_anchor}"
                target_summary = f"<summary>{target_anchor}"
                target_bold = f"**{target_anchor}"
                logging.debug(f"Target header: {target_header}")
                logging.debug(f"Target summary: {target_summary}")
                logging.debug(f"Target bold: {target_bold}")
                logging.debug(f"Target href: {target_href}")
                logging.debug(f"Target ID: {target_id}\n")
                target_source_file = os.path.normpath(os.path.join(directory_name, target_source_file))
                if os.path.isfile(target_source_file):
                    with open(target_source_file, "r") as target_file:
                        target_file_content = target_file.read()
                        if target_header.lower() in target_file_content.lower():
                            break
                        elif target_summary.lower() in target_file_content.lower():
                            break
                        elif target_bold.lower() in target_file_content.lower():
                            break
                        elif target_href.lower() in target_file_content.lower():
                            break
                        elif target_id.lower() in target_file_content.lower():
                            break
                        else:
                            logging.error(f"{filepath} contains a link to '{header_link[0]}#{header_link[1]}', but '{target_anchor}' not found in {target_source_file}\n")
                else: # this is to let us know if we've somehow got an invalid target_source_file
                    logging.error(f"{filepath} contains a link to {header_link}, but {target_source_file} isn't a file.\n")

```
{% endcode %}

This function sets up the logging, calls the `get_md_page_paths` function to get the source files called in `SUMMARY.md`, then loops through those source files.&#x20;

For each source file it's getting the full file path for the file, and also the name of the directory the file is in. It's then calling the `get_header_links_in_pages` function to find all links to section headers and other anchors in the file. If there aren't any, it moves on to the next file. If there are any, it loops over the links found and gets the file being linked to, and the anchor. It does a little bit of transformation on the anchor, and then checks that the file being linked to exists.&#x20;

* If it does, it reads the file being linked to and looks for the anchor in that file. If it finds the anchor, it moves on to the next link. If it doesn't, it outputs an error to the log file.
* If it doesn't exist, the script outputs an error to the log file.

We won't go through every line in this tutorial, but let's take a closer look at some of the parts that may need more explanation.

{% code title="" %}
```python
logging.basicConfig(filename="check_header_links.log", level=logging.ERROR, format="{asctime} - {levelname} - {message}", style="{", datefmt="%Y-%b-%d %H:%M:%S", filemode="w")
```
{% endcode %}

This line sets up logging.&#x20;

* We're going to output errors to a file called `check_header_links.log`. We could change the logging level (`level=logging.ERROR`) to `INFO` or `DEBUG` to capture more information when the script runs. This can be useful when troubleshooting issues in a script, but it can also be too much information to include when running it routinely.
* Each error will output the date and time it was found, formatted as `YYYY-Mon-DD HH:MM:SS` , followed by the logging level , and the message. See [#example-output](finding-broken-links-to-section-headers.md#example-output "mention").

{% code title="" %}
```python
if target_anchor.startswith("id-"): # this is specific to GitBook's docs at the time of writing
                target_anchor = target_anchor.replace("id-","")
if "azure-a-d" in target_anchor: # this is specific to GitBook's docs at the time of writing
                target_anchor = target_anchor.replace("azure-a-d", "azure-ad")
```
{% endcode %}

These lines are for a couple of specific things I noticed in GitBook's docs when I was developing this script. They sometimes have anchors that start with `id-`, which isn't found in the actual header. They also sometimes have anchors that include `azure-a-d`, but the header actually shows `Azure AD`.&#x20;

You likely won't need these exact lines if you're running this script against your own GitBook site source files, but I'm leaving them in as an example of how to handle edge cases.&#x20;

{% code title="" %}
```python
target_href = f'href="{target_anchor}"'
target_id = f'id="{target_anchor}"'
target_anchor = target_anchor.replace('-', ' ')
target_header = f"# {target_anchor}"
target_summary = f"<summary>{target_anchor}"
target_bold = f"**{target_anchor}"
```
{% endcode %}

Sometimes the anchor exists in the source file for a page being linked to, but we need to transform it a bit to find it. If the anchor exists in an `href` or `id`, we want to preserve the `-` characters. Otherwise we want to replace those with spaces. We also want to be able to check if the anchor is in a header, a summary element, or if it has bold formatting.

{% code title="" %}
```python
with open(target_source_file, "r") as target_file:
    target_file_content = target_file.read()
    if target_header.lower() in target_file_content.lower():
        break
    elif target_summary.lower() in target_file_content.lower():
        break
    elif target_bold.lower() in target_file_content.lower():
        break
    elif target_href.lower() in target_file_content.lower():
        break
    elif target_id.lower() in target_file_content.lower():
        break
    else:
        logging.error(f"{filepath} contains a link to '{header_link[0]}#{header_link[1]}', but '{target_anchor}' not found in {target_source_file}\n")
```
{% endcode %}

To account for variations in capitalization, we're going to compare the anchor and the content of the file being linked to in lower case. We're going to check if the anchor exists in a header, in a summary element, in bold, in an `href`, and in an `id`. If we find it in any of these, we stop there. If we don't, we check the next one. If we don't find it in any of these, we log an error that shows the path to the file that the link is in, the link, the anchor, and the file being linked to.

We can use this information to determine if the error is valid.

When I come across an error, the first thing I do is open the page with the link in my browser. I find the link on the page, and I click it to see if it works. If it does, I would try to determine why it's being flagged as an error, and update the script accordingly.

If the error is valid, I then check to see if there's a similar header that should be linked to instead. If there isn't one that's immediately obvious, I open the source file for this page and the page being linked to on GitHub and I use the **Blame** tab to navigate back through the history of each page to see if the link used to be different, and if the header used to exist in the page being linked to but has since changed.

If I can figure out a valid target for the link, I can then fix it. If I can't, I can also see who has worked on those files and reach out to them for assistance, or I can remove the link if it no longer seems necessary.

#### if \_\_name\_\_ == '\_\_main\_\_'

After the `main()` function, there's an if statement:

{% code title="" %}
```python
if __name__ == '__main__':
    main()
```
{% endcode %}

This just means that the the code in the `main()` function will only be executed when the script is run. It won't be executed if this script is imported as a module in another script. It's not necessary to use this in a simple script, but it does add a bit of future-proofing in case we need to import this script ad a module later.

### Running the script

To run this script, I type `python3 check_header_links.py` in the directory where I've saved it.

### Example output

{% code title="check_header_links.log" %}
```log
2026-Jul-14 11:15:14 - ERROR - /home/trike/Projects/public-docs/creating-content/blocks/cards.md contains a link to 'cards.md#card-size', but 'card size' not found in /home/trike/Projects/public-docs/creating-content/blocks/cards.md

2026-Jul-14 11:15:14 - ERROR - /home/trike/Projects/public-docs/collaboration/share.md contains a link to 'member-management/roles.md#the-guest-role', but 'the guest role' not found in /home/trike/Projects/public-docs/collaboration/member-management/roles.md
```
{% endcode %}
