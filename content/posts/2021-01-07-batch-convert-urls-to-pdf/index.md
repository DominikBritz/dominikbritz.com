---
author: Dominik Britz
date: "2021-01-07"
title: PowerShell – Batch Convert URLs to PDF
postSlug: powershell–batch-convert-urls-to-pdf
featured: false
draft: false
tags:
  - powershell
# description:
#   PowerShell – Batch Convert URLs to PDF
comments: true
cover:
  image: images/featured.jpg
  alt: Arrow pointing right
  responsiveImages: true
---

There are lots and lots of articles on the web describing how to save a whole webpage as PDF. They all use, more or less, the browser's ability to print to PDF. 

I needed to convert many URLs at once. Doing that manually for every URL would have been cumbersome, and that's why I automated the process.

The following tools helped me to achieve my goal:
- [LinkKlipper](http://www.codebox.in/products/linkklipper/): a Chrome extension to export all links from a website
- [wkhtmltopdf](https://wkhtmltopdf.org/): a command-line tool that saves a URL to PDF
- [PDF24](https://en.pdf24.org/): my preferred PDF management tool. You can use any tool; it has to support combining multiple PDFs, though.
- PowerShell: to glue everything together

## Collecting All URLs
- Install [LinkKlipper](http://www.codebox.in/products/linkklipper/) in Chrome or Edge
- In the extension's setting, change the output from CSV to TXT
- Browse to the website you want to convert

By clicking on the extension's icon in the menu bar, you can collect all links on the webpage. The webpage I was looking at contained a menu to all subpages, like a sitemap. That allowed me to export all the URLs I was interested in at once.

## Convert Each URL to a PDF File
With the help of the open-source tool [wkhtmltopdf](https://wkhtmltopdf.org/) and PowerShell you can convert every URL in the TXT file from above to a PDF file.

Download [wkhtmltopdf](https://wkhtmltopdf.org/) and install it.

Look at the PowerShell script below, change the variables if needed, and run the script.

```powershell
$sourceFile = "C:\links.txt" # the source file containing the URLs you want to convert
$destFolder = "C:\output" # converted PDFs will be saved here. Folder has to exist.
 
$linkList = get-content $sourceFile
 
foreach ($link in $linkList)
{  
    $outfile = $link -replace '/','-' # replace slashes with dashes
    $date = get-date -Format "yyyy-MM-dd HH-mm-ss" # adding a date to the filename allows for easy sorting later on
    
    $outfile = $date + ' ' + $outfile + '.pdf'
    
    $destFullPath = Join-Path $destFolder $outfile
    & 'C:\Program Files\wkhtmltopdf\bin\wkhtmltopdf.exe'  --disable-smart-shrinking --no-footer-line --no-header-line --no-outline  "$link" "$destFullPath"
}
```

If you're not satisfied with the PDFs' design, have a look at wkhtmltopdf's [command-line arguments](https://wkhtmltopdf.org/usage/wkhtmltopdf.txt).

## Combine PDFs
The last step is optional. But I prefer reading one big PDF over jumping through many different PDFs. Hence, take your PDF tool of choice, mine is PDF24, and combine all created PDFs into one big PDF file.