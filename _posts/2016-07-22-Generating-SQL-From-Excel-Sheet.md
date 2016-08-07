---
layout: post
title: "Generating SQL from an Excel Sheet"
tags: [code-review, vba, excel, sql]
---

You can see my requested code review of my code [here](http://codereview.stackexchange.com/q/135622/47529).

At my work I occasionally have to work on translations for our websites - a
number of our websites pull their text from a database, which has a table of
translations inside of it.  That table looks something like this

{% highlight: sql %}
CREATE TABLE [dbo].[tblResources](
    [lResourceID] [int] IDENTITY(1000,1) NOT NULL,
    [lLocaleID] [int] NOT NULL,
    [txtResourceKey] [varchar](255) NOT NULL,
    [memText] [nvarchar](max) NOT NULL,
    [txtLastModifiedUsername] [varchar](255) NULL,
    [dtLastModifiedDate] [datetime] NULL
);
{% endhighlight %}

As you can see, given a locale and a resource we can retrieve the appropriately
translated text.

We generally would get these translations from our business people in the form
of an Excel spreadsheet, but it would take a lot of work to convert those
to SQL statements, for a few reasons:

1. The resources may or may not already exist, and this may vary between environments
(QA/PROD) and locales.  This is generally something that the business isn't aware of,
and is unmarked on their spreadsheets.
2. The resources aren't always translated for every locale; sometimes we have
certain features that don't get enabled right away/ever in certain locations.
This is generally visible by the lack of a translation, but isn't always immediately
apparent.

As a solution, we've taken to using `MERGE` statements for our scripts.

{% highlight: sql %}
CREATE TABLE #Resources (
    lLocaleID int NOT NULL,
    txtResourceKey varchar(255) NOT NULL,
    memText nvarchar(max) NOT NULL,
    txtLastModifiedUsername varchar(255) NULL
);

DECLARE @US_LOCALE int = 0
    , @UK_LOCALE int = 1
    , @DE_LOCALE int = 2
    , @JP_LOCALE int = 3
    , @IT_LOCALE int = 4
    , @FR_LOCALE int = 5
    , @ES_LOCALE int = 6;

DECLARE @username varchar(255) = 'daniel.obermiller';
INSERT INTO #Resources VALUES(@US_LOCALE, 'supercool.resourcekey', N'cool', @username);
INSERT INTO #Resources VALUES(@UK_LOCALE, 'supercool.resourcekey', N'cool', @username);
INSERT INTO #Resources VALUES(@DE_LOCALE, 'supercool.resourcekey', N'kühl', @username);
INSERT INTO #Resources VALUES(@JP_LOCALE, 'supercool.resourcekey', N'クール', @username);
INSERT INTO #Resources VALUES(@IT_LOCALE, 'supercool.resourcekey', N'fresco', @username);
INSERT INTO #Resources VALUES(@FR_LOCALE, 'supercool.resourcekey', N'frais', @username);
INSERT INTO #Resources VALUES(@ES_LOCALE, 'supercool.resourcekey', N'guay', @username);
GO

USE OurDatabase;
GO

MERGE tblResources AS Target
    USING #Resources AS Source
ON Target.lLocaleID = Source.lLocaleID
    AND Target.txtResourceKey COLLATE DATABASE_DEFAULT = Source.txtResourceKey COLLATE DATABASE_DEFAULT
WHEN MATCHED
    THEN UPDATE SET
        Target.memText = Source.memText,
        Target.txtLastModifiedUsername = Source.txtLastModifiedUsername,
        Target.dtLastModifiedDate = GETDATE()
WHEN NOT MATCHED BY TARGET
    THEN
        INSERT (lLocaleID, txtResourceKey, memText, txtLastModifiedUsername, dtLastModifiedDate)
        VALUES (Source.lLocaleID, Source.txtResourceKey, Source.memText, Source.txtLastModifiedUsername, GETDATE());
GO
{% endhighlight %}

This helps us avoid primary key conflicts and inconsistencies between databases.
Making this transition was a huge help, but it was still a pain to write out all of 
our queries. To solve this, I wrote a series of VBA macros that can run from the 
Excel spreadsheet to generate a SQL file.

A command button is simply added to the top of the spreadsheet. Clicking it triggers a userform,
which just asks what user name should be logged as the last person to edit, what the name of the
output file should be, and which of our (multiple) databases should be updated.

----------------------

Once they've submitted the form, the magic starts. VBA isn't very fun to look at, so if you're
not a fan then avert your eyes now.

{% highlight: vba %}
Private Sub GenerateSQLCommandButton_Click()
    Dim stream As TextStream
    Dim localeIds(0 To 7) As String
    
    InitLocaleIds localeIds
    
    StartStream stream, Me.FileNameTextBox.Value
    
    SetupTempTableVariables stream, localeIds, Me.UsernameTextBox.Value
    WriteResourceKeys stream, localeIds
    WriteMergeStatements stream
    CleanupSql stream
    
    stream.Close
    
    GenerateSqlUserForm.Hide
End Sub
{% endhighlight %}

We delegate our initialization logic to a few pretty straightforward methods, and then
actually generate our SQL in a few stages

1. Setup our temp table, variables, etc
2. Actually write all of our resource keys from the spreadsheet to the script
3. Write all of our merge statements (there might be a lot due to multiple databases, staging sites, etc)
4. Cleanup our resources, commit our transactions, etc, etc

Of those steps the only one that might be of interest is #2.

{% highlight: vba %}
' Format function adapted from http://stackoverflow.com/a/31730589/3076272
Private Sub WriteResourceKeys(stream As TextStream, localeIds() As String)
    Dim resourceText As String, resourceKey As String, insertTemplate As String
    Dim rowCells As Range, colCell As Range
    Dim row As Integer
    
    insertTemplate = "INSERT INTO #Resources VALUES({0}, '{1}', N'{2}', @username);"
    row = 7
    
    With Worksheets(1)
        Do Until .Cells(row, 1).Value2 = ""
            resourceKey = .Cells(row, 1).Value2
            Set rowCells = .Range(GetRange("B", row, "H", row))

            For Each colCell In rowCells.Cells
                resourceText = colCell.Value2
                If Not IsNull(resourceText) And resourceText <> "" Then
                    stream.WriteLine vbTab & Format(insertTemplate, localeIds(colCell.column - 2), resourceKey, resourceText)
                End If
            Next colCell

            row = row + 1
        Loop
    End With
End Sub
{% endhighlight %}

There were a few annoying gotchas - for one, I didn't know about the `Value2` 
property until I did this project - that caused me a lot of hassle. Additionally,
it ended up being a bit of a pain to work with unicode characters - turns out
VBA doesn't handle them as well as it claims to, and requires a bit of fiddling
(that actually doesn't happen here, but in how I setup the stream and a bit in the
`Format` function).

Essentially all we really have to do is go over every row until we run out,
and then add a new insert statement for every one of our locales that has a value.

