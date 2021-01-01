---
layout: post
title: "Moving notes from Google Keep to Joplin"
date: 2021-01-01 00:23:00 +1000
#modified_date: 2020-12-29 17:00:00 +1000
author: Thomas
categories: privacy joplin guide
---
As part of my new years resolution to be more privacy conscious I've decided to try and migrate as many of my Google services to open-source alternatives as possible. After some investigation Joplin seemed to be a suitable alternative to Google Keep, my primary notes manager. Although Google Keep is more for "sticky" notes and Joplin is more like Evernote where it is supposed to hold virtual notebooks, it seemed like a suitable alternative. I liked the in-built synchronisation and end-to-end encryption (E2EE) support and it's nice having multiple notebooks for categorising my notes.

## Moving over

**Before you follow this guide note that the migration script which generates an Evernote backup of your Keep files so you can export them to Joplin is written for Python 3+ and you will need Python 3+ installed on your system.**

That's right, you need a script to convert the Keep data you download to a format that Joplin can import (Evernotes ENEX format) so you can view your notes in Joplin.

A second thing to note is that by default if a note is unnamed then it will simply be titled after the date it was first made instead of the first line or the start of the content of the note. I suppose someone could rewrite the script to detect when a date is a title and if so then rename the title after the beginning of the content... I had ~300 untitled notes that I manually sorted and deleted after importing but this process can be really time consuming.

### Steps

You need to export your notes from Google Keep, fortunately Google provide a service called "Google Takeout" which allows you to export your data many of their services (Keep included). Head over to [https://takeout.google.com/settings/takeout](https://takeout.google.com/settings/takeout) and you'll get a screen that looks similar to this:

You want to make sure you press "Deselect all"
![Takeout Screen 1](/assets/img/moving-notes-from-google-keep-to-joplin/takeout-1.png)

Then scroll down and find "Keep" and tick it
![Takeout Screen 2](/assets/img/moving-notes-from-google-keep-to-joplin/takeout-2.png)

Before finally creating your export with these parameters
![Takeout Screen 3](/assets/img/moving-notes-from-google-keep-to-joplin/takeout-3.png)

You should shortly receive an email with a link to download your data. Google will make you verify your account with your password before your zip/tgz can be downloaded.

Once you have downloaded and extracted your files download a copy of my modified version of the keep-to-enex script from here: [https://gist.github.com/itsjfx/689ae620222240911a3efae33e313b1b](https://gist.github.com/itsjfx/689ae620222240911a3efae33e313b1b). The original version is available here [https://gitlab.com/charlescanato/google-keep-to-evernote-converter](https://gitlab.com/charlescanato/google-keep-to-evernote-converter) but it didn't work for me so I added a modification that allowed it to work.

Put that script in the folder before the `Keep/` folder, which should be in `Takeout/`. After that run the script with `python keep-to-enex.py -o output.enex -f "./Keep"` - where `output.enex` will be the name of the file generated and "output" will be the name of the notebook created from your backup. Feel free to rename it to something such as "Personal.enex" instead.

Now open Joplin and go File -> Import -> ENEX - Evernote Export File (as Markdown) and select your ENEX backup file you just generated.

![Joplin Import](/assets/img/moving-notes-from-google-keep-to-joplin/joplin-import.png)

Once this process is complete a notebook with the same name as your backup will be created and all your notes (and images) will be imported.
