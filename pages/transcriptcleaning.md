---
title: Transcript Cleaning
layout: page
permalink: /technical/transcriptcleaning.html
# Edit the markdown on in this file to describe your collection
# Look in _includes/feature for options to easily add features to the page
---

<nav style="--bs-breadcrumb-divider: url(&#34;data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='8' height='8'%3E%3Cpath d='M2.5 0L1 1.5 3.5 4 1 6.5 2.5 8l4-4-4-4z' fill='currentColor'/%3E%3C/svg%3E&#34;);" aria-label="breadcrumb">
  <ol class="breadcrumb">
    <li class="breadcrumb-item"><a href="#">Home</a></li>
    <li class="breadcrumb-item"><a href="/technical.html">Technical Musings</a></li>
    <li class="breadcrumb-item active" aria-current="page">Transcript Cleaning</li>
  </ol>
</nav>

# Transcript Cleaning

One of the first hurdles that I have come across to starting my data exploration has been the creation and management of transcripts. Because interviews are being conducted and captured through Zoom, I am able to take advantage of their live captioning feature that is driven by Otter.ai and results in the creation of a VTT file.

### About VTT Files

VTT files are common throughout digital video spaces (including things like YouTube) as a way of capturing not only the text but being able to tie it directly to timestamps and other relevant metadata. The general format of the data is that it provides a caption number, a timestamp range for which that caption applies to, and the content of the caption from that period of time. In the case of Zoom, the caption text also includes an attempt to provide information about the speaker, but I have so far found that this is not always accurate and requires human checking.

The fact that there is this much information available pretty much as soon as the interview is over is great. It has certainly saved me a lot of time as compared to if I were starting from just the audio files alone.

### The Issue

VTT files are not generally friendly for human reading or for computer-assisted qualitative data analysis software (CAQDAS), which means that in order to be able to use the files, it is necessary to transform and clean them.

### The Solution - Part I: Converting the File

The first step to this process is transforming the VTT file into a format that is much friendlier for cleaning and refining tools.

After a small amount of searching, I identified Leo Zhou's <a href="https://gist.github.com/glasslion/b2fcad16bc8a9630dbd7a945ab5ebf5e">VTT to Text python script</a> on GitHub, which serves this purpose well.

Running the files through this script creates a tab separated, though still messy, version of the transcripts in `.txt` form.

### The Solution - Part II: Cleaning the Data

These versions of the documents are closer to what one would need, but still not perfect. It is necessary to add some additional structure for them to be useable in spreadsheet or dataframe form, a goal for my purposes as I am interested in further cleaning the data using R before ingesting the transcripts either into <a href="https://ropenscilabs.github.io/qcoder/index.html">qcoder</a> or the much less built-for-purpose <a href="https://uidaholib.github.io/oral-history-as-data/">Oral History as Data</a> tool, both of which are very lightweight, open-source tools that will enable me to code the data but not get entrenched behind paywalls as would be the case with NViVo or restricted by the printing costs of physical/manual coding.

I went down a few different paths when it came to attempting to transform the transcripts from their txt state and ultimately found that the following combination of tools and workflow resulted in the least complicated and most effective way to prepare the data for ingestion.

#### Tools

The primary tool that enabled me to prepare the data was <a href="https://openrefine.org/">OpenRefine</a>. I found that it was far more effective than trying to start the process a spreadsheet software such as Excel.

However, OpenRefine does have its own limitations, and so in order to be able to fully format the transcripts as I wanted, I did need to use a spreadsheet editor after having performed the bulk of the data cleaning in order to be able to concatenate lines at the cell level.

#### Process

After converting a from VTT to TXT, I directly imported the TXT file into OpenRefine. Because the file was tab separated to an extent, the import is just clean enough to start using OpenRefines splitting and separation functions.

###### Step 1. Split text from timestamps

Start by splitting the imported data on a regex that captures the "SPEAKER:" element of the transcripts — `(.*):`. This will help to create a split in the information provided by the columns into a text column and a timestamp column. It does however remove the information about the speaker from the data.

###### Step 2. Recapture speaker metadata

To recapture this information into a useable column, you can then create a new column based on the original text that uses a jython `ifelse` clause to identify and return the various possible speakers (the general format of the clause being if "NAME" in value: return "NAME"; this is also useful for de-identification as it allows you to supply the de-identifier as the 'return' value). As mentioned above, the AI isn't perfect at assigning this, so you will need to do some manual checking.

###### Step 3. Isolate Timestamps

While the timestamps have now been separated from the text of the transcript, they are still in the time range form utilized by VTT files (i.e. 00:00:58.050 --> 00:01:09.660). Since the range isn't particularly important for coding purposes, the starting timestamp can be isolated by splitting the values using `-->` as the delimiter.

###### Step 4. Remove Caption Numbers

Just as the AI is not perfect, the script and OpenRefine are not perfect in making it possible to isolate lines as lines; for example, the VTT file contains designated caption identifier numbers that, in the conversion, are treated as text. Sometimes these numbers get assigned their own row, sometimes they get grouped together with the timestamp, and sometimes they get grouped with the text, so to address this, I performed a variety of actions to isolate and remove these numbers.

Here are details of the various workflows:

<b>For instances where the numbers were assigned their own row</b>, if you've used the "guess cell format" option, you can facet by whether the cell format is numeric and remove the those rows.

<b>For those that are within the same cell as a necessary timestamp</b>, I split the timestamp column, this time using a single space as the delimiter (thankfully there isn't anything else that uses it in that chunk of the data!).

The split of the timestamp column helps to isolate the starting timestamps, but it still creates a bit of a mess because everything is in multiple columns. The best thing about OpenRefine in this process is the ability to use facets to manipulate and clean the data. In this case, it is possible to facet the relevant columns by whether or not they are a numeric value, clear out the numeric 'row' values, then isolate blank values in the desired column and transpose information from the split columns that will be deleted. To do this replacement, you can use OpenRefine's transform functions and a GREL command that identifies the value of the relevant cell (`cells['secondcolumnname'].value`).

<b>For those that are within the same cell as text</b>, I ran the following GREL expression on the text column: `replace(value,/\d/,"")`, which uses RegEx to remove all integers. You will need to manually check the transcript for any numbers such as ages or dates that get removed because of this process, but ultimately that is easier than trying to go in and remove all of the caption numbers since you'll likely be reading through and having to correct things within the AI generated transcript text anyway.

###### Step 5. Filling in missing data from rows 
Because of the way the columns were split, even though all timestamps will be in one column now, there won't be a text row that has a timestamp attached to it (and also no timestamp row that has text). In order to assign timestamps to text rows, you can use the OpenRefine fill down command to populate any cells that don't have a timestamp value in them. After, it is then possible to facet by whether or not the 'text' column has content and remove those that don't.

###### Step 7. Concatenating text
The next step for cleaning the data is a little trickier. Because of the way VTTs are built, multiple sentences get split across multiple rows and sometimes even a single sentence will be multiple rows. For the purposes of coding, it is far more useful to have the transcripts chunked into larger pieces that reflect trains of thought. This isn't something that can be done just by asking a program to combine rows. Instead, I found two ways to address this need. The first is rather tiresome, which is just to copy and paste the relevant text from cell to cell, which requires clicking a lot of 'edit' buttons because of how OpenRefine is built. The slightly faster way is to export the document to a spreadsheet editor and make use of a  `CONCATENATE` function that will paste multiple cell contents together, but still requires a fair amount of lot of copy and pasting, as well as selecting and deleting action in order to remove the rows that are no longer necessary.
    
    NOTE: While faster, the second option is still not ideal because it means removing the file from OpenRefine and needing to re-enter it in order finish cleaning the document. This ends up creating a whole new document because as as far as I am aware, OR doesn't have an option for versioning in such a way that you can make changes outside of OR and then update the file within the program.

Process:

1. CONCATENATE all cells corresponding to a single speaker (any time there is a value in the speaker column)
2. Make sure to capture the values rather than the formulas of the newly generated column (can be as easy as just copying and using paste special into another column)
2. Re-introduce the data to OpenRefine
3. Facet for empty speaker cells (since any text from a row with an empty speaker is now concatenated) and delete those cells
4. At this point there will still be some cells in the original text column that contain unique information because the concatenation will not have affected single-line responses. In order to make sure that all text is captured within one column, facet for empty cells in the concatenated text column and fill them with the corresponding cell value from the original text column. You can do this by using the same GREL expression as before (`cells['secondcolumnname'].value`).
5. Delete the 'original' text column. You should now have one timestamp column, one speaker column, and one text column.

###### Step 8. Join speaker attribution to text

The last step that one might perform in OpenRefine is an optional one depending on where you plan to use your data next. For me, it is important at this point to retain a separate 'speaker' column because of the program that I am using for qualitative analysis. However, this is something that you could join with the interview text should that be better for your purposes.

### Conclusion 

This process feels as though it is a little more brute force than it is an elegant solution to the issue, but I find that it still helps to make my transcripts feel far more manageable for future cleaning and ingestion.
