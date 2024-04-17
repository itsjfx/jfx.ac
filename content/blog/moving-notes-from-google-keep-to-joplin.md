---
layout: post
title: Moving notes from Google Keep to Joplin
date: 2021-01-01
tags: [privacy, joplin, guide]
---

## Introduction

As part of my new years resolution to be more privacy conscious, I've tried to
migrate as many of my Google services to open-source alternatives as possible.
After some investigation Joplin seemed to be a suitable alternative to Google
Keep, my primary notes manager.

Although Google Keep is more for "sticky" notes and Joplin is more like Evernote
where it is supposed to hold virtual notebooks, it's a pretty good alternative.
I like the in-built synchronisation and end-to-end encryption (E2EE) support,
and it's nice having multiple notebooks for categorising my notes.

## Guide

### Beware

* The migration script generates an Evernote formatted backup of your Keep files
  which Joplin can import, and requires Python 3+
* If a note is unnamed then it'll be tilted after the date it was first created
* If you have lots of unnamed notes this might take some time to clean up

### Steps

To export your notes from Google Keep Google offer a service called "Google
Takeout" which allows you to export your data many of their services.

1. Head to [Google Takeout](https://takeout.google.com/settings/takeout):
    1. Click `Deselect all`
    2. Find `Keep` and tick it
    3. Under the next step, select `Send download link via email`, `Export Once`
       for frequency, with the highest file size
2. You should shortly receive an email with a link to download your data. Google
   will make you verify your account with your password before your zip/tgz can
   be downloaded.
3. Once you have downloaded and extracted your files download a copy of my
modified version of the keep-to-enex script from here:
    * <https://gist.github.com/itsjfx/689ae620222240911a3efae33e313b1b>
    * The original version is available here
      <https://gitlab.com/charlescanato/google-keep-to-evernote-converter>
      * but it didn't work for me so I added a modification that allowed it to
        work
4. Put that script in the folder before the `Keep/` folder, which should be in
`Takeout/`. 
5. Run with `python keep-to-enex.py -o output.enex -f "./Keep"` where
   `output.enex` will be the name of the file generated and "output" will be the
   name of the notebook created from your backup.
6. Open Joplin and go File -> Import -> ENEX - Evernote Export File (as
   Markdown) and select your ENEX backup file you just generated.
7. A notebook with the same name as your backup will be created and all your
   notes (and images) will be imported.
