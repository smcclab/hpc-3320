# COMP3320 Scrape for sharing

The idea of the repo is to translate the .mbz moodle backup of COMP3320 High Performance COmputing into a portable git repo that can be shared with potential teachers (outside ANU so can't access the old Moodle or new Wattle page).

The .mbz is a zip file I believe.

I'd like to produce a very simple structured repo with dirs for `lectures`, `labs`, `resources`, `assessments`.

An example of how I have done this before is here: https://charlesmartin.au/blog/2026/03/30/pandoc-course-template

There should be HTML files with a course schedule and links to all the necessary files and resources. I imagine that a lot of PDF files and perhaps even video files are in the mbz. I don't know, you'll have to look. You can use Bootstrap to make the HTML files (and resulting site) presentable.

Things you DON'T have to do:

- turn PDFs into editable text files: keep the PDFs, just link to them.
- translate Moodle quizzes/other special interactive elements into something. A placeholder or link to whatever weird source will be sufficient.

Things you DO have to do:

- arrange the files nicely so that the potential lecturer can access what they need.
- make sure all the live course content is available.
- make sure that this can be easily served. Directory full of files with html docs would be a good bet. Something that can be served locally to explore or put on the web without much effort.




