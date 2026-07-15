---
layout: post
title: "Syncing TFS with a local directory"
date: 2013-06-19 23:38:38 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2013/06/19/syncing-tfs-with-a-local-directory/
---

I started learning PowerShell in order to automate some of the more tedious activities in TFS. For example, I needed to treat a local directory as the source of truth instead of the version in source control. I came across [this blog post](http://techblog.dorogin.com/2011/09/tfs-how-to-sync-server-folder-with.html) that provides a script that seemed to do exactly what I needed. The first time I executed this script on a directory with over 10,000 sub-directories and files, it took 10 minutes to finish.

Here is phase 1 of the script:

{% raw %}
```powershell
# Phase 1: add all local files into TFS which aren't under source control yet
$items = Get-ChildItem -Recurse 

foreach($item in $items) {
	$localItem = $item.FullName
	$serverItem = Get-TfsChildItem -Item "$localItem"
	
	if (!$serverItem -and !($pendingAdds -contains $localItem)) {
		# if there's no server item AND there's no a pending Add
		write "No such item as '$localItem' on the server, adding"
		Add-TfsPendingChange -Add "$localItem"
	}
}
```
{% endraw %}

With over 10,000 files in the local directory, calling `Get-TfsChildItem` for each item wasn't time efficient. Running phase 1 took approximately 3 minutes.

Looking at the help page for `Get-TfsChildItem`, we may pass an array `QualifiedItemSpec[]` instead of a single item. Instead of calling `Get-TfsChildItem` thousands of time, we can call it just once.

{% raw %}
```powershell
# Phase 1: add all local files into TFS which aren't under source control yet
$items = Get-ChildItem -Recurse | % { $_.FullName }

$serverItems = Get-TfsChildItem -Item $items

foreach($serverItem in $serverItems){
   if (!$serverItem -and !($pendingAdds -contains $localItem)) {
      write "No such item as '$localItem' on the server"
   }
}
```
{% endraw %}

Running the updated script still took approximately 2 minutes. It isn't much of an improvement, but it is better nonetheless. Phase 2 of the script has the same problem.

{% raw %}
```powershell
# Phase 2: delete all subfolder/files in TFS if there's no local subfolder/file for them anymore, and check out other items
$items = Get-TfsChildItem -Recurse

foreach($item in $items) {
   $serverItem = $item.ServerItem
   $localItem = Get-TfsItemProperty -Item $serverItem 
   
   # Do other stuff...
}
```
{% endraw %}

For the entire collection of files found in TFS, the script will iterate through and call `Get-TfsItemProperty` for each file. Running phase 2 took approximately 6 minutes. Again, we can update the script to call `Get-TfsItemProperty` once by passing it an array.

{% raw %}
```powershell
# Phase 2: delete all subfolder/files in TFS if there's no local subfolder/file for them anymore, and check out other items
$items = Get-TfsChildItem -Recurse
$itemProperties = Get-TfsItemProperty -Item $items

foreach($item in $itemProperties) {
   $serverItem = $item.SourceServerItem
   $localItem = $item.LocalItem
   
   # Do other stuff...
}
```
{% endraw %}

Running the updated script still took approximately 3 minutes. Better, but still slow. After reading the help page for each cmdlet more carefully, I finally noticed that most of these cmdlets offer the `-Recurse` switch. Instead of iterating through each of the files in the local workspace and server, we can use the top directory along with the `-Recurse` switch.

{% raw %}
```powershell
$localItems = Get-ChildItem -Recurse | % { $_.FullName }
$serverItemProperties = Get-TfsItemProperty -Item . -Recurse
$serverItems = $serverItemProperties | % { $_.LocalItem }

# Phase 1: add all local files into TFS which aren't under source control yet
foreach ($item in $localItems) {
   if (!($serverItems -contains $item) -and !($pendingAdds -contains $item)) {
      write "No such item as '$localItem' on the server, adding"
      Add-TfsPendingChange -Add "$item"
   }  
}

# Phase 2: delete all subfolder/files in TFS if there's no local subfolder/file for them anymore, and check out
foreach($item in $serverItemProperties) {  
	$serverItem = $item.SourceServerItem
	$localItem = $item.LocalItem
	
	# Do other stuff...
}
```
{% endraw %}

The entire script runs in just 16 seconds instead of the original 10 minutes.
