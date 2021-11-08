---
layout: post
title:  "Security Issues on FusionPBX"
description: This article explains the research made by me and my friends at FusionPBX.
tags: CVE
---
In this research, I and two friends have found that FusionPBX is vulnerable to various security issues, but in this article, I will focus on the 4 vulnerabilities that got assigned as CVE. These vulnerabilities are:

1. CVE-2021-43403
2. CVE-2021-43404
3. CVE-2021-43405
4. CVE-2021-43406

So let's start from the beginning! The first vulnerability, assigned as CVE-2021-43403 is a path traversal found in the file **"app/log_viewer/log_viewer.php"**.

As you can see in this piece of code, it was passing the GET parameter **n**, to the function fopen, which is been passed to the function fpassthru, consequently opening the file passed in the user input.

```php
if (isset($_GET['n']) && substr($_GET['n'],0,14) == "freeswitch.log") {
	$dir = $_SESSION['switch']['log']['dir'];
	$filename = $_GET['n'];
	session_cache_limiter('public');
	$fd = fopen($dir."/".$filename, "rb");
	header("Content-Type: binary/octet-stream");
	header("Content-Length: " . filesize($tmp."/".$filename));
	header('Content-Disposition: attachment; filename="'.$filename.'"');
	fpassthru($fd);
	exit;
}
```
But, as can you see in line 1, the first 14 characters need to be **"freeswitch.log"**, we can possibly "bypass" this with another vulnerability to create a folder called **"freeswitch.loganything"**, and using a payload like `"freeswitch.loganything/../../../../../../etc/passwd"` to successfully read the system files.

Do you can found the patch of this vulnerability [here][patch-0].

The next 2 security failures that I will explain are very similar, for two main reasons: They two are in the same file and they two are command injection.

A malicious attacker can use the fact that the fax_extension parameter is used later in the function exec without any type of filter or treatment, enabling a malicious attacker to inject commands into the server.
```php
if (strlen($_REQUEST["fax_extension"]) > 0) {
	$fax_extension = $_REQUEST["fax_extension"];
}
...
$command = $IS_WINDOWS ? '' : 'export HOME=/tmp && ';
$command .= 'libreoffice --headless --convert-to pdf --outdir '.$dir_fax_temp.' '.$dir_fax_temp.'/'.$fax_name.'.'.$fax_file_extension;
exec($command);
```
Do you can found the patch of this vulnerability [here][patch-1].

The last vulnerability is basically the same as the last one, the fax_page_size parameter is used without any filter or treatment in the exec function, making it possible for a malicious attacker to inject commands into the server.
```php
$fax_page_size = $_POST['fax_page_size'];
...
$cmd = 'tiff2pdf -u i -p '.$fax_page_size.
	' -w '.$page_width.
	' -l '.$page_height.
	' -f -o '.
correct_path($dir_fax_temp.'/'.$fax_instance_uuid.'.pdf').' '.
correct_path($dir_fax_temp.'/'.$fax_instance_uuid.'.tif');
exec($cmd);
```
Do you can found the patch of this vulnerability [here][patch-2].

I would like to thank my friends @silvanojr and @carlos_crowsec, who made this research in collaboration with me.

[patch-0]: https://github.com/fusionpbx/fusionpbx/commit/57b7bf0d6b67bda07d550b07d984a44755510d9c
[patch-1]: https://github.com/fusionpbx/fusionpbx/commit/2d2869c1a1e874c46a8c3c5475614ce769bbbd59
[patch-2]: https://github.com/fusionpbx/fusionpbx/commit/0377b2152c0e59c8f35297f9a9b6ee335a62d963